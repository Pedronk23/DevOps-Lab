# OpenBao / Vault — Alta disponibilidad (HA) y Raft

**OpenBao** es el fork open source de HashiCorp Vault, ahora bajo la Linux Foundation.

**HA** (*High Availability*): desplegar varias instancias del servidor para que el gestor de
secretos no tenga un único punto de fallo. Si OpenBao se cae y todos los secretos viven en él,
se para todo: HA existe para evitar ese problema.

## Cómo funciona en OpenBao/Vault: modelo activo-pasivo

- Tiene varios nodos en un clúster.
- **Un solo nodo es el líder (active)** y es el único que atiende lecturas y escrituras en un
  momento dado.
- Los demás son **standby (pasivos)**. Están arrancados y sincronizados, pero solo esperando.
  Si les llega una petición, la reenvían al líder.
- Si el líder se cae, los standby celebran una **elección de líder** y uno de ellos toma el
  relevo automáticamente. A esto se le llama **failover**.

```
                      clientes
                         │
                 Service openbao (8200)
            ┌────────────┼────────────┐
            ▼            ▼            ▼
      ┌──────────┐ ┌──────────┐ ┌──────────┐
      │openbao-0 │ │openbao-1 │ │openbao-2 │
      │ ACTIVE   │ │ standby  │ │ standby  │
      │ (líder)  │ │          │ │          │
      └────┬─────┘ └────▲─────┘ └────▲─────┘
           │  replica   │            │
           └── log Raft ┴────────────┘   (puerto de clúster 8201)

   petición a un standby ──► se reenvía al active
```

> Vault Enterprise tiene *performance standbys* que sí sirven lecturas. En la edición libre la
> regla general es activo-pasivo: revisa la documentación de tu versión antes de contar con
> escalado de lecturas.

## Coordinar quién es el líder: el backend de almacenamiento

**Raft (Integrated Storage)**
El propio OpenBao replica los datos entre los nodos usando el algoritmo de consenso Raft.
No necesita nada externo. Es la opción recomendada hoy en día.
Normalmente quieres un **número impar de nodos (3 o 5)** para poder alcanzar quorum en las
votaciones.

**Backend externo** (antiguamente Consul, etc.)
El estado y la coordinación viven fuera. Más piezas móviles.

## Raft en 30 segundos

```
   1. Un nodo es LÍDER; los demás, SEGUIDORES.
   2. Toda escritura va al líder, que la añade a su log.
   3. El líder replica la entrada a los seguidores.
   4. Cuando la MAYORÍA confirma → la entrada se "comitea".
   5. Si los seguidores dejan de oír al líder → elección:
      gana quien consigue votos de la mayoría.
```

### Quorum: por qué impar

Quorum = mayoría estricta = `(N / 2) + 1` (división entera).

| Nodos | Quorum | Fallos que aguanta |
|---|---|---|
| 1 | 1 | 0 |
| 2 | 2 | 0 ← peor que 1: el doble de máquinas y la misma tolerancia |
| 3 | 2 | **1** |
| 4 | 3 | 1 ← no mejora a 3 |
| 5 | 3 | **2** |
| 7 | 4 | 3 (más latencia de escritura; raro necesitarlo) |

> Un nodo par no añade tolerancia a fallos, solo coste y más votos que coordinar.

## Configuración de un nodo

```hcl
# /openbao/config/config.hcl
ui = true

listener "tcp" {
  address         = "[::]:8200"         # API y UI
  cluster_address = "[::]:8201"         # tráfico entre nodos
  tls_cert_file   = "/openbao/tls/tls.crt"
  tls_key_file    = "/openbao/tls/tls.key"
}

storage "raft" {
  path    = "/openbao/data"
  node_id = "openbao-0"

  retry_join {
    leader_api_addr = "https://openbao-0.openbao-internal:8200"
  }
  retry_join {
    leader_api_addr = "https://openbao-1.openbao-internal:8200"
  }
  retry_join {
    leader_api_addr = "https://openbao-2.openbao-internal:8200"
  }
}

api_addr     = "https://openbao-0.openbao-internal:8200"
cluster_addr = "https://openbao-0.openbao-internal:8201"
```

| Puerto | Uso |
|---|---|
| **8200** | API, CLI y UI (lo que usan los clientes) |
| **8201** | tráfico de clúster: replicación Raft y reenvío de peticiones |

## En Kubernetes

```
   StatefulSet openbao  (replicas: 3)
     ├── openbao-0  + PVC data-openbao-0
     ├── openbao-1  + PVC data-openbao-1
     └── openbao-2  + PVC data-openbao-2

   Service openbao-internal   (headless → DNS estable por pod, para Raft)
   Service openbao            (todos los pods)
   Service openbao-active     (solo el pod con la etiqueta de líder)
   Service openbao-standby    (solo los standby)
```

Valores típicos de la chart de Helm:

```yaml
server:
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
  affinity: |
    podAntiAffinity:            # cada réplica en un nodo distinto
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app.kubernetes.io/name: openbao
          topologyKey: kubernetes.io/hostname
```

> Sin anti-afinidad, las 3 réplicas pueden acabar en el mismo nodo de Kubernetes: si ese
> nodo cae, cae todo el "HA".

## Sealed / unsealed en HA

Cuando un nodo arranca está **sealed** (sellado): tiene los datos cifrados pero no puede
descifrarlos hasta que se le proporcione una clave.

En un escenario HA, **cada nodo que arranca o toma el relevo necesita estar unsealed para poder
servir**. En despliegues serios se usa **auto-unseal** (ver
[static seal e init](02-openbao-seal-e-init.md)).

```
   init  → SOLO en el primer nodo (openbao-0)
   join  → los demás se unen al clúster Raft existente
   unseal→ en CADA nodo (automático con static seal / KMS)
```

## Comandos de operación

```bash
# estado del nodo (HA Mode: active / standby)
bao status

# miembros del clúster Raft y quién es el líder
bao operator raft list-peers
```

```
Node         Address                        State       Voter
----         -------                        -----       -----
openbao-0    openbao-0.openbao-internal:8201  leader      true
openbao-1    openbao-1.openbao-internal:8201  follower    true
openbao-2    openbao-2.openbao-internal:8201  follower    true
```

| Comando | Para qué |
|---|---|
| `bao operator raft join https://openbao-0.openbao-internal:8200` | unir un nodo a mano (si no hay `retry_join`) |
| `bao operator raft remove-peer openbao-2` | sacar un nodo muerto del clúster |
| `bao operator step-down` | forzar que el líder actual ceda el liderazgo (mantenimiento) |
| `bao operator raft autopilot state` | salud de los nodos vista por autopilot |
| `bao operator raft snapshot save backup.snap` | **copia de seguridad** completa |
| `bao operator raft snapshot restore backup.snap` | restaurar |

```bash
# backup desde fuera del pod
kubectl exec -n mi-namespace openbao-0 -- \
  bao operator raft snapshot save /tmp/openbao.snap
kubectl cp mi-namespace/openbao-0:/tmp/openbao.snap ./openbao-$(date +%F).snap
```

> El snapshot va cifrado con la barrera: para restaurarlo necesitas **también** el seal
> (la clave estática o el KMS) con el que se creó.

## Escenarios de fallo

| Situación | Qué pasa |
|---|---|
| Cae un standby (de 3) | nada visible; quorum 2/3 intacto |
| Cae el líder (de 3) | elección en segundos; uno de los standby pasa a active |
| Caen 2 de 3 | **sin quorum**: no hay líder, no se puede leer ni escribir |
| Partición de red 1 / 2 | el lado con 2 nodos sigue; el nodo aislado no puede ser líder |
| Se pierde el PVC de un nodo | se borra el peer, se recrea el pod y vuelve a unirse |

Recuperar un clúster que ha perdido el quorum de forma permanente se hace con un fichero
`peers.json` en `<path>/raft/peers.json` con los nodos supervivientes. Es una operación de
último recurso: primero intenta devolver los nodos caídos.

## Errores típicos

| Síntoma | Causa habitual |
|---|---|
| `local node not active but active cluster node not found` | no hay líder: quorum perdido o nodos sin unseal |
| `failed to join raft cluster` | DNS del Service headless, TLS (el certificado no incluye `openbao-X.openbao-internal`) o puerto 8201 bloqueado |
| Un nodo aparece dos veces en `list-peers` | se recreó con otro `node_id` sin quitar el antiguo: `remove-peer` |
| Todo va lento al escribir | discos lentos o nodos en zonas lejanas: Raft espera a la mayoría |
