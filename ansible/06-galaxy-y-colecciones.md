# Ansible — Galaxy privado y público (colecciones)

## 1. Qué problema resuelve Galaxy

Escribimos **colecciones** de Ansible (`miorg.utils`, `miorg.roadrunner`, …). Una colección
es básicamente un paquete: un `.tar.gz` con roles, módulos y playbooks dentro.

Si queremos distribuir esas colecciones necesitamos un **sitio central** donde subir los
paquetes y poder descargarlos. Ese sitio central es un **registro de colecciones**, el mismo
concepto que:

| Registro | Para |
|---|---|
| npm | JavaScript |
| PyPI | Python |
| Play Store | apps Android |
| **Galaxy** | Ansible |

## 2. Dos tipos de Galaxy

| Tipo | Detalle |
|---|---|
| **Galaxy público** | lo ve todo el mundo: `galaxy.ansible.com`, en internet, de Red Hat. Software: Galaxy NG |
| **Galaxy privado** | solo lo ve quien queremos; se aloja en un servidor propio, dentro de la red interna. Software: Galaxy NG |

En nuestro caso lo tenemos privado: en algún sitio se ejecuta el propio Galaxy NG.

## 3. Qué hay dentro del servidor

```
┌─────────────────────────────────────────┐
│ SERVIDOR                                │
│  ┌───────────────────────────────────┐  │
│  │ GALAXY NG (web, login, botones)   │  │ ← la cara bonita: web + API
│  └───────────────────────────────────┘  │
│  ┌───────────────────────────────────┐  │
│  │ PULP (guarda los .tar.gz de       │  │ ← el motor de almacén por debajo
│  │ verdad)                           │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

> Si en la URL aparece `.../pulp/api/v3/...` estamos hablando directamente con el motor de
> almacén.

## 4. Dentro de Pulp: los "repositorios"

Pulp no guarda todo en un montón: lo organiza en **repositorios** (cajones). El cajón típico
se llama `published` ("lo publicado y aprobado").

Cada repositorio tiene un interruptor `private` sí/no. Es exactamente la casilla
*"Make this repository private"* de la web.

## 5. Cómo entran y salen las colecciones

**Cómo ENTRAN (publicar)**: el pipeline de CI, en la etapa de deploy, hace:

1. Construye el `.tar.gz` de la colección.
2. Lo sube al servidor con `ansible-galaxy collection publish`, autenticándose con un token.

**Cómo SALEN (instalar / consumir)**: quien quiera usar la colección ejecuta:

```bash
ansible-galaxy collection install miorg.utils
```

Pero Ansible necesita saber **a qué servidor preguntar**: eso está configurado en el fichero
`ansible.cfg` (sección `[galaxy]`, `server_list`).

## 6. Parte privada / pública

Para cambiar el acceso a público hay que cambiar dos ajustes en el servidor:

```
GALAXY_ENABLE_UNAUTHENTICATED_COLLECTION_ACCESS = True   # (ver colecciones)
GALAXY_ENABLE_UNAUTHENTICATED_COLLECTION_DOWNLOAD = True # (descargarlas)
```

## 7. Estos ajustes no están en la web

Son ajustes de configuración del servidor: viven en un fichero del sistema operativo.

- `/etc/pulp/settings.py`, o como variables de entorno con prefijo `PULP_…`
- O en el inventario del instalador, si es un AAP / Automation Hub "oficial" de Red Hat.

> Tras tocarlos hay que **reiniciar los servicios**. Para eso necesitas entrar al servidor
> (por SSH) o pedirlo a quien lo administre.

## Estructura de una colección

```
miorg-utils-1.4.2.tar.gz
└── ansible_collections/miorg/utils/
    ├── galaxy.yml            ← metadatos: nombre, versión, dependencias
    ├── README.md
    ├── plugins/
    │   ├── modules/          módulos propios
    │   ├── filter/           filtros Jinja2
    │   └── action/           action plugins
    ├── roles/
    │   └── iis/
    ├── playbooks/
    └── meta/runtime.yml      versión mínima de ansible-core
```

```yaml
# galaxy.yml
namespace: miorg
name: utils
version: 1.4.2
readme: README.md
authors: [equipo-devops]
dependencies:
  "ansible.windows": ">=2.0.0"
```

## Ciclo de vida completo

```
   desarrollo            CI (GitLab/GitHub)              consumo
  ┌──────────┐        ┌─────────────────────┐        ┌──────────────┐
  │ git push │──────► │ lint + molecule     │        │ requirements │
  │ tag 1.4.2│        │ ansible-galaxy      │        │ .yml         │
  └──────────┘        │   collection build  │        └──────┬───────┘
                      │ ansible-galaxy      │               │
                      │   collection publish│──► Galaxy ────┘
                      └─────────────────────┘    privado
                                                (Galaxy NG + Pulp)
```

## requirements.yml

Así se declara lo que consume un proyecto, en vez de instalar a mano:

```yaml
collections:
  - name: miorg.utils
    version: ">=1.4.0,<2.0.0"
  - name: ansible.windows
    version: 2.5.0

roles:
  - name: geerlingguy.nginx
    version: 3.1.4
```

```bash
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml --force
```

## Configurar el servidor en ansible.cfg

```ini
[galaxy]
server_list = mi_galaxy, galaxy_publico

[galaxy_server.mi_galaxy]
url = https://galaxy.interno.local/api/galaxy/content/published/
token = <token>           # mejor por variable de entorno

[galaxy_server.galaxy_publico]
url = https://galaxy.ansible.com/
```

> El orden de `server_list` importa: Ansible pregunta en ese orden y usa la primera respuesta.

## Comandos

```bash
ansible-galaxy collection init miorg.utils       # esqueleto de colección
ansible-galaxy collection build                  # genera el .tar.gz
ansible-galaxy collection publish miorg-utils-1.4.2.tar.gz --server mi_galaxy
ansible-galaxy collection install miorg.utils:==1.4.2
ansible-galaxy collection list                   # qué hay instalado y dónde
```

## Versionado semántico

```
   1  .  4  .  2
   │     │     └── patch: correcciones, sin cambios de interfaz
   │     └──────── minor: nuevas funcionalidades compatibles
   └────────────── major: rompe compatibilidad
```

Regla práctica en `requirements.yml`: fijar el major (`>=1.4.0,<2.0.0`) para recibir mejoras
sin que un cambio incompatible te rompa el pipeline.
