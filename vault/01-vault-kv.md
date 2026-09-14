# HashiCorp Vault — Motor KV

`vault kv` (*Key-Value*) es uno de los motores de secretos de Vault.

## Dónde encaja el motor KV

```
   CLIENTE (tú, un pod, Ansible, CI)
      │
      │ 1. se autentica (token, userpass, kubernetes, approle…)
      ▼
   ┌──────────── VAULT / OPENBAO ─────────────────────────────┐
   │  auth method ──► token ──► policies (qué rutas puede ver)│
   │                               │                          │
   │                               ▼                          │
   │   secrets engines montados en rutas:                     │
   │     secret/     → KV v2        (contraseñas, API keys)   │
   │     pki/        → PKI          (certificados)            │
   │     database/   → credenciales dinámicas de BBDD         │
   │     transit/    → cifrado como servicio                  │
   └──────────────────────────────────────────────────────────┘
```

- **Motor de secretos** (*secrets engine*): un "plugin" montado en una ruta. KV es el más
  simple: guarda pares clave-valor tal cual.
- Todo lo que hay debajo de la ruta de montaje (`secret/…`) lo gestiona ese motor.

## Versiones del motor

| Versión | Comportamiento |
|---|---|
| **v1** | siempre sobrescribe, no guarda versiones |
| **v2** | versionado: se puede consultar una versión concreta o la `latest` |

```bash
vault secrets list -detailed            # ver motores montados y su versión (columna Options)
vault secrets enable -path=secret kv-v2 # montar un KV v2 en secret/
vault kv enable-versioning secret/      # convertir un KV v1 existente en v2
```

## Conectarse

```bash
export VAULT_ADDR="https://vault.lab.local:8200"
export VAULT_CACERT="/etc/ssl/certs/ca-lab.pem"   # si la CA es interna
vault login                                        # pide el token
vault login -method=userpass username=pedro        # otro método de auth
vault token lookup                                 # quién soy, TTL y policies
```

> Con OpenBao el CLI es `bao` y las variables `BAO_ADDR`/`BAO_TOKEN`; la sintaxis de
> `kv` es la misma (`bao kv get …`).

## Comandos

| Comando | Qué hace |
|---|---|
| `vault kv put` | crea o actualiza un secreto |
| `vault kv get` | lee un secreto |
| `vault kv delete` | elimina un secreto |
| `vault kv list` | lista los secretos disponibles en una ruta |
| `vault kv patch` | actualiza solo algunos campos de un secreto |

### Ejemplos

```bash
# crear (put REEMPLAZA el secreto entero: los campos que no pases desaparecen)
vault kv put secret/apps/web usuario=app password=s3cr3t

# añadir o cambiar un campo sin perder el resto
vault kv patch secret/apps/web puerto=5432

# leer
vault kv get secret/apps/web
vault kv get -field=password secret/apps/web          # solo un campo (ideal para scripts)
vault kv get -format=json secret/apps/web | jq -r '.data.data.usuario'

# evitar que la contraseña quede en el historial de la shell
vault kv put secret/apps/web password=-               # "-" = leer el valor de stdin
vault kv put secret/apps/web @datos.json              # "@" = cargar desde fichero
```

## Navegar por el árbol de secretos

Se va bajando nivel a nivel con `list`:

```bash
vault kv list secret
vault kv list secret/equipo
vault kv list secret/equipo/prod
vault kv list secret/equipo/prod/app-web
```

### Regla para distinguir carpeta de secreto

- Si el elemento **acaba en `/`** → es un **directorio**, se abre con `list`.
  Ej.: `equipo/`
- Si **no acaba en `/`** → es el **secreto** en sí, se accede con `get`.
  Ej.: `lectura`

```bash
vault kv get secret/equipo/prod/app-web/lectura
# → accedemos al secreto
```

```
secret/
└── equipo/                      ← list
    └── prod/                    ← list
        └── app-web/             ← list
            ├── lectura          ← get  (secreto)
            └── escritura        ← get  (secreto)
```

## Versionado en KV v2

```
   put v1 ──► put v2 ──► put v3 (latest)
                │
                └── vault kv get -version=2 …   ← leer una versión antigua
```

| Comando | Qué hace | ¿Recuperable? |
|---|---|---|
| `vault kv delete secret/x` | borrado **lógico** de la última versión | sí, con `undelete` |
| `vault kv undelete -versions=3 secret/x` | recupera versiones borradas | — |
| `vault kv destroy -versions=2 secret/x` | borra **físicamente** esas versiones | no |
| `vault kv metadata delete secret/x` | borra el secreto con **todas** sus versiones y metadatos | no |
| `vault kv rollback -version=2 secret/x` | crea una versión nueva con el contenido de la 2 | — |

```bash
vault kv metadata get secret/apps/web                     # versiones, fechas, borrados
vault kv metadata put -max-versions=10 secret/apps/web    # limitar histórico
vault kv put -cas=3 secret/apps/web password=nuevo        # check-and-set: solo si la actual es la v3
```

> `-cas=0` significa "escribe solo si el secreto **no existe**": útil para no pisar valores
> en scripts de bootstrap.

## Rutas reales de la API (clave para las policies)

El CLI oculta un detalle: en **KV v2** la API inserta `data/` o `metadata/` en la ruta.

```
CLI:      vault kv get secret/apps/web
API:      GET /v1/secret/data/apps/web
                        ^^^^
CLI:      vault kv list secret/apps
API:      LIST /v1/secret/metadata/apps
                        ^^^^^^^^
```

Por eso una policy para KV v2 se escribe así:

```hcl
# policy: app-web-lectura.hcl
path "secret/data/apps/web" {
  capabilities = ["read"]
}

path "secret/metadata/apps/*" {
  capabilities = ["list"]
}
```

```bash
vault policy write app-web-lectura app-web-lectura.hcl
vault token capabilities secret/data/apps/web      # qué puede hacer mi token en esa ruta
```

| Capability | Permite |
|---|---|
| `read` | `get` |
| `create` / `update` | `put`, `patch` |
| `delete` | `delete` |
| `list` | `list` |
| `sudo` | rutas protegidas del sistema |
| `deny` | niega explícitamente (gana a todo) |

## Uso desde otras herramientas

```yaml
# Ansible (colección community.hashi_vault)
- name: Leer contraseña de la app
  ansible.builtin.set_fact:
    db_password: "{{ lookup('community.hashi_vault.vault_kv2_get', 'apps/web',
                     engine_mount_point='secret')['secret']['password'] }}"
```

```bash
# en un script
export DB_PASSWORD="$(vault kv get -field=password secret/apps/web)"
```

## Errores típicos

| Error | Causa habitual |
|---|---|
| `permission denied` (403) | la policy apunta a `secret/apps/web` en vez de `secret/data/apps/web` |
| `no handler for route "secret/…"` | no hay motor montado en esa ruta, o la ruta de montaje es otra |
| `preflight capability check returned 403` | el token no puede leer `sys/internal/ui/mounts`: pasa `-mount=secret` |
| `No value found at secret/data/…` | la ruta no existe o la última versión está borrada |
| El `put` "borró" campos | `put` reemplaza el secreto entero: para un solo campo usa `patch` |
| `x509: certificate signed by unknown authority` | falta `VAULT_CACERT` con la CA interna |

## Buenas prácticas

- Estructura de rutas pensada para policies: `secret/<equipo>/<entorno>/<app>`.
- **Nunca** escribir secretos como argumentos en claro: `password=-` o `@fichero`.
- Tokens de corta duración para personas; AppRole o auth de Kubernetes para máquinas.
- `max-versions` razonable: el histórico también es superficie de fuga.
