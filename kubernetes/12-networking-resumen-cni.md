# Kubernetes Networking (3) — Resumen: Ingress, Services, CNI y pods

```
                    KUBERNETES NETWORKING
        ┌──────────────┬──────────────┬──────────────┐
        ▼              ▼              ▼              ▼
    INGRESS        SERVICES       CNI PLUGIN       PODS
```

## Ingress

Expone HTTP y HTTPS.

**Características:**
- Terminación SSL/TLS.
- URLs externas.
- Enrutamiento basado en path (*path based routing*).

**Ingress Controller** — implementaciones:
- nginx
- Traefik
- Cilium
- En cloud: AGIC (Azure), ALB Ingress Controller (AWS), etc.

Se configura mediante un **Ingress Resource** (manifiesto YAML).

## Services

Un Service ofrece una **dirección consistente** para acceder a un conjunto de pods.

**Por qué se necesitan:**
- Los pods son efímeros.
- Los pods cambian constantemente de IP.
- El sistema necesita una forma de seguirles la pista.

**Tipos:**

| Tipo | Detalle |
|---|---|
| **ClusterIP** | el valor por defecto. Crea una IP interna del clúster para el servicio |
| **NodePort** | expone un puerto en cada nodo, permitiendo acceso desde fuera |
| **LoadBalancer** | se usa con cloud providers para enrutar tráfico externo hacia el clúster |

## CNI plugin (Container Networking Interface)

Provee la red de conectividad a los contenedores:

- Configura las interfaces de red en los contenedores.
- Asigna direcciones IP y configura las rutas (via iptables en los nodos).

**Plugins habituales**: Cilium, Calico, Flannel.

## Pods (networking a nivel de pod)

- Cada pod obtiene su propia dirección IP.
- Por defecto, todos los pods pueden conectar con todos los pods de todos los nodos.
- Los contenedores dentro de un mismo pod se comunican entre sí por `localhost`.

## Flujo completo de una petición externa

```
Usuario                     
(app.ejemplo.com)
    │
    ▼
   DNS  ──────────────►  LoadBalancer de Kubernetes
(proveedor DNS/cloud)     creado por nginx o Traefik
  (p.ej. 10.0.0.14)       puertos: 80 / 443
                                   │
                                   ▼
                               INGRESS
                                   │  (routing rule)
                                   ▼
                               SERVICE
                                ╱     ╲
                            POD         POD
```
