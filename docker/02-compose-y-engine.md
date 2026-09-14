# Docker — Compose y Docker Engine

## Ejemplo: Voting App (multi-contenedor)

```
┌──────────────┐                                    ┌───────────────┐
│ Vote         │                                    │ Result        │
│ (Python)     │                                    │ (Node.js)     │
└──────┬───────┘                                    └───────▲───────┘
       ▼                                                    │
┌──────────────┐      ┌──────────────┐      ┌───────────────┴───────┐
│ redis        │─────►│ worker       │─────►│ Base de datos         │
│ (cola)       │      │ (.NET)       │      │ (PostgreSQL)          │
└──────────────┘      └──────────────┘      └───────────────────────┘
                    todo definido en compose.yaml
```

En Docker "a mano" estos servicios se enlazaban con `--link` (obsoleto); hoy se declaran en un
`compose.yaml` y Docker crea una red propia donde **cada servicio es resoluble por su nombre**.

## compose.yaml

```yaml
services:
  vote:
    build: ./vote
    ports:
      - "5000:80"
    depends_on:
      - redis
    environment:
      REDIS_HOST: redis          # resuelve por el nombre del servicio

  redis:
    image: redis:7-alpine

  worker:
    build: ./worker
    depends_on:
      - redis
      - db

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 5

volumes:
  db-data:
```

> `depends_on` solo controla el **orden de arranque**, no espera a que el servicio esté listo.
> Para eso hace falta un `healthcheck` con `depends_on: {db: {condition: service_healthy}}`.

## Comandos de Compose

```bash
docker compose up -d                 # levantar en background
docker compose -f otro-fichero.yml up
docker compose ps
docker compose logs -f worker
docker compose exec db psql -U postgres
docker compose down                  # parar y borrar contenedores y red
docker compose down -v               # …y también los volúmenes (borra datos)
docker compose build --no-cache
```

## Docker Engine (Linux)

Tres piezas:

| Pieza | Función |
|---|---|
| **Docker daemon** | gestiona imágenes, contenedores, volúmenes, redes… |
| **REST API** | para que programas, herramientas e integraciones hablen con el daemon |
| **Docker CLI** | los comandos que escribes (`docker run`, `docker stop`) |

**Acceso remoto**: TLS en el puerto 2376. Recomendado: hacerlo por SSH
(`DOCKER_HOST=ssh://usuario@servidor`), que no expone el daemon a la red.

> Exponer el socket del daemon sin TLS equivale a dar root en el host: quien pueda hablar con
> el daemon puede montar `/` dentro de un contenedor privilegiado.

## Cadena de ejecución

```
docker CLI → docker DAEMON → containerd → runc
   │              │              │          │
 comando        API, red,     runtime    crea el proceso usando
 que escribes   volúmenes    (estándar)  namespaces y cgroups del
                                          kernel de Linux
```

## Almacenamiento

```
   ┌──────────── contenedor ────────────┐
   │  capa escribible (se pierde al rm) │
   └───────┬───────────────┬────────────┘
           │               │
     -v volumen      --mount type=bind
           │               │
   /var/lib/docker/    ruta del host
     volumes/...       (p. ej. /datos)
```

- `/var/lib/docker`: toda la información de Docker (imágenes, volúmenes, contenedores).
- **Volúmenes** (gestionados por Docker, la opción recomendada para datos):

```bash
docker volume create datavolume
docker run -v datavolume:/var/lib/mysql mysql
docker volume ls
docker volume inspect datavolume
```

- **Bind mounts** con `--mount` (sintaxis moderna y explícita), útiles en desarrollo para
  montar tu código dentro del contenedor:

```bash
docker run --mount type=bind,source=/datos,target=/var/lib/mysql mysql
```

| | Volumen | Bind mount |
|---|---|---|
| Lo gestiona | Docker | tú (ruta del host) |
| Portabilidad | alta | depende de la ruta |
| Uso típico | datos de BBDD, producción | código en desarrollo |

## Redes

| Driver | Para qué |
|---|---|
| `bridge` | por defecto: red privada en el host, con DNS por nombre de contenedor |
| `host` | el contenedor usa la red del host, sin aislamiento (máximo rendimiento) |
| `none` | sin red |
| `overlay` | red entre varios hosts (Swarm) |

```bash
docker network create mi-red
docker run -d --network mi-red --name db postgres
docker run -it --network mi-red alpine ping db     # resuelve por nombre
```

## Orquestación

- **Docker Swarm**: el orquestador propio de Docker, sencillo pero con poca adopción.
- **Kubernetes**: el estándar de facto hoy (ver la carpeta [kubernetes](../kubernetes/)).

```
  docker run          →   compose        →   orquestador
  1 contenedor            varios en 1 host    varios hosts, reprogramación,
                                              escalado y autorreparación
```
