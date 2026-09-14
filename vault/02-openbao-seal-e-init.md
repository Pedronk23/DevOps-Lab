# OpenBao — Barrier, seal e init

## Conceptos

**La barrera (barrier)**: todo lo guardado en OpenBao está cifrado en disco con una **clave
maestra**. Esa clave es el corazón de la caja fuerte.

**El seal (sello)**: es cómo se protege y se abre esa clave maestra. Con un **static seal**, la
clave maestra queda "envuelta" por una clave estática que vive en un Secret de Kubernetes
(p. ej. `bao-seal-key`). Al arrancar el pod, OpenBao lee esa clave estática,
desenvuelve la maestra y se **auto-desella** (*unseal* automático). Por eso no se meten unseal
keys a mano.

**init (inicializar)**: el acto de crear la caja fuerte por primera vez. Genera la clave
maestra, monta la barrera de cifrado y crea el primer usuario todopoderoso (**root token**).
Sin `init` = OpenBao en blanco.

> ⚠️ El **static seal NO crea la caja fuerte**, solo sabe abrirla. `init` es quien la crea.

## Las capas de claves

```
   ┌───────────────────────────────────────────────┐
   │  SEAL  (Shamir / static / KMS / transit)      │  ← quién abre la caja
   │   └── desenvuelve ▼                           │
   │  ROOT KEY  (antes "master key")               │
   │   └── descifra ▼                              │
   │  KEYRING  (claves de cifrado, rotables)       │
   │   └── cifra/descifra ▼                        │
   │  DATOS en el storage (Raft, fichero…)         │  ← siempre cifrados en disco
   └───────────────────────────────────────────────┘
```

- **Sealed**: el proceso está arrancado pero no tiene la root key en memoria → no puede leer
  nada. Solo responde a `status` y a operaciones de unseal.
- **Unsealed**: la root key está en memoria → sirve peticiones.
- Si alguien copia el disco o el volumen, **solo ve datos cifrados**.

## Tipos de seal

| Seal | Cómo se desella | Qué genera el init | Uso |
|---|---|---|---|
| **Shamir** (por defecto) | a mano: `bao operator unseal` con N trozos de clave | **unseal keys** | labs, o donde no hay KMS |
| **Static** | automático, con una clave estática (fichero o variable) | **recovery keys** | Kubernetes on-premise sin KMS |
| **KMS cloud** (AWS KMS, Azure Key Vault, GCP KMS) | automático, pidiendo a la nube que desenvuelva | **recovery keys** | nube |
| **Transit** | automático, contra otro OpenBao/Vault | **recovery keys** | "un Vault que abre a otro" |

```
   Shamir (manual)                        Auto-unseal (static / KMS)
   ───────────────                        ──────────────────────────
   arranca → sealed                       arranca → lee clave del seal
   operador mete clave 1/3                         → desenvuelve root key
   operador mete clave 2/3                         → unsealed, sin intervención
   operador mete clave 3/3 → unsealed
   (y así en CADA reinicio de CADA nodo)
```

## Comando de inicialización

```bash
kubectl exec -n mi-namespace openbao-0 -- \
  bao operator init --recovery-shares=1 --recovery-threshold=1
```

Desglose:

| Parte | Qué hace |
|---|---|
| `kubectl exec -n mi-namespace openbao-0 --` | entra en el pod `openbao-0` del namespace indicado y ejecuta lo que viene después. Todo lo posterior al `--` se ejecuta dentro del contenedor |
| `bao operator init` | subcomando que crea la caja fuerte desde cero |
| `--recovery-shares` / `--recovery-threshold` | cuando el seal es automático (static), el init no genera *unseal keys* sino **recovery keys**; estos dos flags controlan cómo se reparten |

- `--recovery-shares=n`: en cuántos trozos se parte la recovery key (algoritmo de Shamir).
- `--recovery-threshold=m`: cuántos trozos hacen falta para reconstruirla.

### Shamir en una imagen

```
                 recovery key
                      │
       ┌──────┬───────┼───────┬──────┐
       ▼      ▼       ▼       ▼      ▼
     trozo1 trozo2 trozo3  trozo4 trozo5      shares = 5
       │      │       │
       └──────┴───┬───┘
                  ▼
          con 3 cualesquiera                  threshold = 3
          se reconstruye la clave
```

| Entorno | shares / threshold |
|---|---|
| Laboratorio | `1 / 1` (cómodo, sin separación de funciones) |
| Producción | `5 / 3`, repartidos entre personas distintas |

## Qué devuelve el init

| Salida | Uso |
|---|---|
| **Recovery Key 1** | en la práctica no se usa. Se guarda por si acaso (regenerar el root token en el futuro) |
| **Initial Root Token** | esta sí es la importante. Es el usuario todopoderoso recién creado. Se necesita para el re-bootstrap: engine KV, auth de Kubernetes, policies y roles |

Para guardarlo de forma automatizable, en JSON:

```bash
kubectl exec -n mi-namespace openbao-0 -- \
  bao operator init -recovery-shares=1 -recovery-threshold=1 -format=json > init.json

jq -r '.root_token' init.json
jq -r '.recovery_keys_b64[0]' init.json
```

> `init.json` contiene las llaves del reino: muévelo a su custodia (Secret, gestor de
> contraseñas) y **bórralo** del disco. Nunca a Git (el `.gitignore` del repo ya excluye
> `*secret*` y similares, pero no confíes solo en eso).

## Comprobar el estado

```bash
kubectl exec -n mi-namespace openbao-0 -- bao status
```

```
Key                      Value
---                      -----
Seal Type                static
Recovery Seal Type       shamir
Initialized              true         ← ¿se hizo el init?
Sealed                   false        ← ¿está abierta?
Total Recovery Shares    1
Threshold                1
HA Enabled               true
HA Mode                  active       ← active / standby
```

| Comando | Código de salida |
|---|---|
| `bao status` | `0` unsealed · `2` sealed · `1` error |
| `bao operator init -status` | `0` inicializado · `2` sin inicializar · `1` error |

Útil en scripts de arranque: "si `init -status` devuelve 2, haz el init".

## Importante

Los valores antiguos (root token y recovery keys de la vez anterior) **quedan obsoletos en
cuanto haces el init**. Toca actualizar los Secrets de custodia (`bao-root-token` y
`bao-recovery-keys`) con los nuevos.

**Unseal keys**: no sirven para desellar (porque es automático), pero sí para **recuperar el
root token** si se pierde.

> Matiz de nombres: con auto-unseal lo que devuelve el init son **recovery keys**, aunque
> coloquialmente se las siga llamando "unseal keys". Son las que se usan en `generate-root`.

```bash
# actualizar los Secrets de custodia tras un init nuevo
kubectl -n mi-namespace create secret generic bao-root-token \
  --from-literal=token="$(jq -r '.root_token' init.json)" \
  --dry-run=client -o yaml | kubectl apply -f -
```

## Operaciones del día después

| Comando | Para qué |
|---|---|
| `bao operator generate-root -init` | empezar a regenerar un root token (pide recovery keys) |
| `bao token revoke <root-token>` | revocar el root token cuando acabas el bootstrap |
| `bao operator seal` | sellar a mano (emergencia: corta el servicio al instante) |
| `bao operator rotate` | rotar la clave de cifrado del keyring |
| `bao operator rekey -target=recovery` | cambiar las recovery keys (o su reparto) |

```
   Flujo recomendado del root token
   ────────────────────────────────
   init → root token → bootstrap (KV, auth, policies, roles)
                          │
                          ▼
                    revocar el root token
                          │
          ¿hace falta otra vez? → generate-root con recovery keys
```

## Errores típicos

| Síntoma | Causa habitual |
|---|---|
| `Vault is sealed` / `503` | el seal no puede leer su clave (Secret mal montado o renombrado) |
| `security barrier not initialized` | nunca se hizo el `init` (o se borró el volumen de datos) |
| `Vault is already initialized` | intentas un `init` sobre datos existentes |
| El pod no pasa a Ready | la readiness probe de la chart exige `initialized` + `unsealed` |
| Root token "no vale" | es el de un init anterior: los datos se recrearon |

## Buenas prácticas

- La clave del seal y las recovery keys **no deben vivir juntas**: quien tenga ambas lo tiene
  todo.
- No dejes el root token vivo "por comodidad": revócalo y regénéralo cuando haga falta.
- Copia de seguridad de la clave del static seal: sin ella, los datos cifrados son
  irrecuperables.
- Documenta quién custodia cada trozo de recovery key.
