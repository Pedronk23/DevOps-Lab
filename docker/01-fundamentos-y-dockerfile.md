# Docker — Fundamentos e imágenes

## Contenedor vs máquina virtual

```
      MÁQUINAS VIRTUALES                    CONTENEDORES
 ┌──────┐ ┌──────┐ ┌──────┐            ┌──────┐ ┌──────┐ ┌──────┐
 │ app  │ │ app  │ │ app  │            │ app  │ │ app  │ │ app  │
 ├──────┤ ├──────┤ ├──────┤            ├──────┴─┴──────┴─┴──────┤
 │ S.O. │ │ S.O. │ │ S.O. │  ← pesado  │   container runtime    │
 ├──────┴─┴──────┴─┴──────┤            ├────────────────────────┤
 │       hipervisor        │            │  kernel del host       │  ← compartido
 ├────────────────────────┤            ├────────────────────────┤
 │       hardware          │            │       hardware         │
 └────────────────────────┘            └────────────────────────┘
   arranque: minutos                      arranque: milisegundos
```

Un contenedor es un **proceso aislado** del host mediante funciones del kernel de Linux:
*namespaces* (aislamiento de vista: PID, red, montajes) y *cgroups* (límite de recursos).

## Evolución de los runtimes

```
LXC            →   libcontainer   →   runc            →   containerd
(early Docker)     (2014, 0.9)        (estándar OCI)      (hoy)
```

- **LXC**: los primeros Docker usaban LXC (*Linux Containers*) por debajo.
- **libcontainer**: Docker 0.9 (2014) lo sustituye por su propia librería, escrita en Go.
- **runc**: libcontainer se dona a la **OCI** (*Open Container Initiative*) y se convierte en
  `runc`, el runtime de referencia del estándar.
- **containerd**: gestiona el ciclo de vida completo (imágenes, snapshots, ejecución) y delega
  en `runc`. Es lo que usan hoy Docker y Kubernetes, este último sin Docker de por medio.

## Docker en Windows

Docker es tecnología del kernel de Linux (namespaces y cgroups). En Windows, Docker Desktop
levanta una máquina virtual Linux por debajo:

```
   contenedor Linux  →  VM Linux (WSL2 o Hyper-V)  →  host Windows
```

Por eso el rendimiento del disco y las rutas de los *bind mounts* no se comportan igual que
en Linux.

## Ejecutar contenedores

```bash
docker run -d nginx        # -d: corre en el background (detached)
```

- **Docker Hub**: registro público de imágenes por defecto. `docker run nginx` equivale a
  `docker.io/library/nginx:latest`.
- **Tag**: la versión de la imagen (`docker run redis:7.4`). Sin tag se usa `latest`.
- Un contenedor **no está pensado para correr todo un sistema operativo**, solo una función:
  cuando acaba la función, muere.
- Un contenedor base no abre una terminal para enviar información de vuelta. Para eso:

| Flag | Qué hace |
|---|---|
| `-i` | *interactive*: mantiene STDIN abierto (permite responder a un script) |
| `-t` | asigna una pseudo-terminal |
| `-it` | modo terminal interactivo |

Otros flags que se usan a diario:

```bash
docker run -d --name web -p 8080:80 nginx        # publicar puerto host:contenedor
docker run --rm -it ubuntu bash                  # se borra al salir
docker run -e ENTORNO=prod mi-app                # variable de entorno
docker run --restart=unless-stopped mi-app       # política de reinicio
```

## Ciclo de vida de un contenedor

```
  docker run
      │
      ▼
  ┌─────────┐  docker pause   ┌────────┐
  │ running │────────────────►│ paused │
  └────┬────┘◄────────────────└────────┘
       │        docker unpause
       │ docker stop (SIGTERM → SIGKILL)
       ▼
  ┌─────────┐  docker start   
  │ exited  │──────────────► running
  └────┬────┘
       │ docker rm
       ▼
    borrado
```

## Imágenes y capas

Cada instrucción del Dockerfile crea una **capa** inmutable. Las capas se cachean: si una
cambia, se reconstruyen todas las de abajo.

```
  ┌───────────────────────────┐  capa escribible (el contenedor)
  ├───────────────────────────┤
  │ ENTRYPOINT flask run      │  ┐
  ├───────────────────────────┤  │
  │ COPY app.py /opt/         │  │  capas de la imagen
  ├───────────────────────────┤  │  (solo lectura, compartidas
  │ RUN apt-get install flask │  │   entre contenedores)
  ├───────────────────────────┤  │
  │ FROM ubuntu               │  ┘
  └───────────────────────────┘
```

> Regla de oro del caché: pon primero lo que cambia poco (dependencias) y al final lo que
> cambia siempre (tu código). Si copias el código antes de instalar dependencias, invalidas
> el caché en cada build.

## Instrucciones para montar una imagen (Dockerfile)

```dockerfile
FROM ubuntu                              # sistema operativo base
RUN apt-get update                       # actualizar repos de paquetes
RUN apt-get install -y python3-flask     # instalar Flask
COPY app.py /opt/app.py                  # copiar el código del servicio
ENV FLASK_APP=/opt/app.py                # variable de entorno
ENTRYPOINT flask run --host=0.0.0.0      # arrancar el servidor web
```

```bash
docker build -t webapp .     # construir la imagen con el Dockerfile del directorio actual
docker init                  # genera Dockerfile, compose.yaml y .dockerignore según el lenguaje
```

### Instrucciones más usadas

| Instrucción | Para qué |
|---|---|
| `FROM` | imagen base |
| `RUN` | ejecuta un comando **en build** y crea una capa |
| `COPY` | copia archivos del contexto a la imagen |
| `ADD` | como COPY pero además descomprime y acepta URLs (mejor usar COPY) |
| `ENV` | variable de entorno persistente |
| `ARG` | variable **solo** en build (`--build-arg`) |
| `WORKDIR` | directorio de trabajo |
| `EXPOSE` | documenta el puerto (no lo publica) |
| `USER` | usuario con el que corre el proceso |
| `VOLUME` | punto de montaje declarado |
| `ENTRYPOINT` | el ejecutable principal |
| `CMD` | argumentos por defecto del ENTRYPOINT (o comando si no hay ENTRYPOINT) |

### ENTRYPOINT vs CMD

```
ENTRYPOINT ["ping"]        CMD ["localhost"]
      │                          │
      └──── docker run img  → ping localhost
            docker run img google.com → ping google.com   (CMD se sustituye)
```

Dependiendo del ordenador donde se construya, la imagen puede ser **amd64** o **arm64**.

## Cuatro problemas del build clásico

1. Los paquetes se re-descargan en cada build (mal aprovechamiento de caché).
2. *Secrets leak*: los secretos acaban en capas de la imagen.
3. *Architecture lock-in*: la imagen queda atada a la arquitectura de quien la construyó.
4. *Sequential stages*: las etapas se ejecutan en secuencia, sin paralelismo.

## Solución: BuildKit

Ya viene activado en las versiones modernas de Docker. Para lo específico, `buildx`:

```bash
docker buildx build -t web .
docker buildx build --platform linux/amd64,linux/arm64 -t web:1.0 --push .
docker build --secret id=token,src=./token.txt -t web .   # secreto que no queda en capas
```

En el Dockerfile, ese secreto solo existe mientras dura el `RUN`:

```dockerfile
RUN --mount=type=secret,id=token cat /run/secrets/token
```

## Multi-stage build

Resuelve el problema del tamaño: compilas en una imagen gorda y copias solo el binario a una
mínima.

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN go build -o /app

FROM gcr.io/distroless/static
COPY --from=build /app /app
ENTRYPOINT ["/app"]
```

```
  [ imagen build: 900 MB ]  →  copia el binario  →  [ imagen final: 15 MB ]
```

## Buenas prácticas

- Tags explícitos (`nginx:1.27`), nunca `latest` en producción.
- `.dockerignore` para no mandar `.git`, `node_modules` ni secretos al contexto de build.
- Un proceso por contenedor.
- Correr como usuario no root (`USER 1000`).
- Agrupar `RUN` con `&&` y limpiar el caché de paquetes en la misma capa.

## Comandos de inspección

```bash
docker ps -a                 # contenedores, también los parados
docker images
docker logs -f web
docker exec -it web sh
docker inspect web           # JSON completo: red, montajes, variables
docker stats                 # consumo en vivo
docker system df             # cuánto espacio ocupa todo
docker system prune -a       # limpiar (¡borra imágenes sin usar!)
```

> Los IDs no hace falta escribirlos enteros: bastan los primeros caracteres si son únicos
> (`docker stop d1ba`). Y `docker ps --no-trunc` los muestra completos, sin recortar.
