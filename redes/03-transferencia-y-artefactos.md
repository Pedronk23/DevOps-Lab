# Redes — Transferencia de archivos y repositorios de artefactos

## curl

En consola, sirve entre otras cosas para saber si una página web está activa:

```bash
curl -I https://google.com
```

`-I` pide solo las cabeceras (método HEAD), así no descarga el cuerpo de la respuesta.

### Opciones que más se usan

| Opción | Qué hace |
|---|---|
| `-I` | solo cabeceras (HEAD) |
| `-L` | sigue redirecciones (301/302) |
| `-v` | muestra toda la conversación: DNS, TLS, cabeceras enviadas y recibidas |
| `-s` / `-S` | silencioso / pero muestra errores (`-sS` juntos) |
| `-f` | falla con código de salida ≠ 0 si el HTTP es ≥ 400 (clave en scripts) |
| `-o fichero` / `-O` | guarda en un fichero / con el nombre remoto |
| `-X POST` | método HTTP |
| `-H "Cabecera: valor"` | añade una cabecera |
| `-d '{"a":1}'` | cuerpo de la petición |
| `-u usuario:pass` | autenticación básica |
| `-k` | ignora errores de certificado (**solo para diagnosticar**) |
| `--cacert ca.pem` | confía en una CA concreta (la alternativa buena a `-k`) |
| `--resolve host:443:IP` | fuerza la IP sin tocar DNS ni `/etc/hosts` |
| `-x http://proxy:3128` | usar un proxy |
| `--connect-timeout 5` / `-m 30` | timeout de conexión / total |
| `-w '%{http_code}\n'` | imprime datos de la respuesta (código, tiempos…) |

### Recetas

```bash
# código HTTP y nada más (health checks en scripts)
curl -s -o /dev/null -w '%{http_code}\n' https://app.lab.local/health

# llamar a una API con JSON y token
curl -sS -f -X POST https://api.lab.local/v1/deploy \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"app":"web","version":"1.4.2"}'

# probar un Ingress por nombre antes de que exista el DNS
curl -v --resolve web.lab.local:443:192.168.1.240 https://web.lab.local/

# ¿dónde se va el tiempo? (DNS, TCP, TLS, primer byte)
curl -s -o /dev/null -w 'dns:%{time_namelookup} tcp:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total}\n' https://app.lab.local

# descargar con reintentos
curl -fL --retry 3 -o app.tar.gz https://nexus.lab.local/repository/raw/app/1.0/app.tar.gz
```

> En Windows 10+ existe `curl.exe` real. En Windows PowerShell 5.1, `curl` a secas es un alias
> de `Invoke-WebRequest`: escribe `curl.exe` para usar el de verdad.

`wget` es la alternativa clásica para descargar: `wget -q https://…/fichero`,
`wget -c` para continuar una descarga cortada.

### Códigos HTTP que hay que reconocer

| Código | Significado | Pista en DevOps |
|---|---|---|
| `200` / `201` / `204` | OK / creado / sin contenido | — |
| `301` / `302` / `308` | redirección | falta `-L`, o redirección HTTP→HTTPS |
| `400` | petición mal formada | JSON inválido, cabecera mal |
| `401` | **no autenticado** | falta el token o es inválido |
| `403` | **autenticado pero sin permiso** | policy, RBAC, IP no permitida |
| `404` | no existe | ruta mal, o el Ingress no casa con el host |
| `413` | cuerpo demasiado grande | límite del proxy (`client_max_body_size` en nginx) |
| `429` | demasiadas peticiones | *rate limit* |
| `500` | error interno de la aplicación | logs de la app |
| `502` | Bad Gateway | el proxy no consigue hablar con el backend (pod caído, puerto mal) |
| `503` | servicio no disponible | sin endpoints sanos (readiness fallando) |
| `504` | Gateway Timeout | el backend tarda más que el timeout del proxy |

## FTP (File Transfer Protocol)

Protocolo de red estándar que permite transferir archivos entre un cliente y un servidor
a través de Internet o de una red local.

**Cómo funciona**: el cliente se conecta al servidor FTP (normalmente en el puerto 21) y
puede subir, descargar, mover o eliminar archivos.

FTP usa **dos conexiones**: una de control (21) y otra de datos. Eso complica los firewalls:

```
   MODO ACTIVO                              MODO PASIVO (el habitual hoy)
   ───────────                              ─────────────────────────────
   cliente ──► :21  control                 cliente ──► :21  control
   cliente ◄── :20  datos                   cliente ──► :50000-51000 datos
   (el SERVIDOR abre conexión               (el cliente abre las dos: pasa
    hacia el cliente: choca con NAT)         mejor NAT, pero hay que abrir
                                              el rango pasivo en el firewall)
```

Variantes:

| Protocolo | Qué es |
|---|---|
| **FTPS** | FTP con cifrado SSL/TLS |
| **SFTP** | transferencia de archivos sobre SSH (más seguro y más común hoy en día) |

| | FTP | FTPS | SFTP |
|---|---|---|---|
| Base | TCP | FTP + TLS | SSH |
| Puertos | 21 + datos | 21 (explícito, `AUTH TLS`) o 990 (implícito) + datos | **solo 22** |
| Cifrado | ninguno (usuario y contraseña en claro) | sí | sí |
| Firewall | complicado | aún más (el firewall no puede inspeccionar el canal cifrado) | trivial |

> Salvo que un tercero lo imponga, usa **SFTP** o `rsync` sobre SSH
> (ver [SCP y SFTP](../seguridad/02-scp-y-sftp.md)).

```bash
rsync -avz --progress ./build/ devops@srv01:/opt/app/     # solo copia lo que cambia
rsync -avz --delete ./build/ devops@srv01:/opt/app/       # espejo exacto (borra lo que sobra)
```

## Repositorios de artefactos (ej. Nexus)

Herramienta de gestión de repositorios de artefactos. Funciona como un **servidor
centralizado** donde los equipos de desarrollo almacenan y distribuyen paquetes de software.
Se accede por **HTTP/HTTPS**.

**Qué guarda**: paquetes Maven (Java), NPM (JavaScript), imágenes Docker, etc.

**Para qué sirve**:

- Actuar como **proxy** entre tu equipo y los repositorios públicos (caché, disponibilidad).
- Publicar artefactos privados de la organización sin exponerlos al público.
- Controlar quién puede subir y descargar paquetes.

### Tipos de repositorio

```
                         ┌─────────── GROUP  (npm-all) ───────────┐
   cliente (npm, pip,    │  una sola URL que agrupa varios repos  │
   docker, maven) ──────►│                                        │
                         │   ┌──────────────┐  ┌────────────────┐ │
                         │   │ HOSTED       │  │ PROXY          │ │
                         │   │ npm-internal │  │ npm-proxy      │─┼──► registry.npmjs.org
                         │   │ (lo nuestro) │  │ (caché)        │ │
                         │   └──────────────┘  └────────────────┘ │
                         └────────────────────────────────────────┘
```

| Tipo | Qué es |
|---|---|
| **Hosted** | almacena lo que **tú publicas** (librerías internas, releases) |
| **Proxy** | caché de un repositorio público: si Internet o el repositorio público caen, sigues construyendo |
| **Group** | agrupa varios bajo una URL; los clientes solo configuran esa |

### Formatos habituales

| Formato | Cliente | Ejemplo de uso |
|---|---|---|
| Maven | `mvn`, Gradle | JARs, WARs |
| npm | `npm`, `yarn` | paquetes JavaScript |
| PyPI | `pip` | paquetes Python |
| Docker | `docker`, `podman` | imágenes de contenedor |
| Helm | `helm` | charts de Kubernetes |
| Raw | `curl` | cualquier fichero: binarios, tar.gz, ISOs |
| Go | `go` | módulos (`GOPROXY`) |
| NuGet | `dotnet`, `nuget` | paquetes .NET |

### Configurar los clientes

```ini
# pip  →  ~/.config/pip/pip.conf  (Windows: %APPDATA%\pip\pip.ini)
[global]
index-url = https://nexus.lab.local/repository/pypi-all/simple
```

```bash
# npm
npm config set registry https://nexus.lab.local/repository/npm-all/

# Docker (el repositorio Docker de Nexus suele ir en su propio puerto o subdominio)
docker login registry.lab.local
docker tag app:1.4.2 registry.lab.local/equipo/app:1.4.2
docker push registry.lab.local/equipo/app:1.4.2

# subir un fichero a un repositorio raw
curl -fu "$NEXUS_USER:$NEXUS_PASS" --upload-file app-1.4.2.tar.gz \
  https://nexus.lab.local/repository/raw-releases/app/1.4.2/app-1.4.2.tar.gz
```

Usar Nexus como mirror de Docker Hub en `/etc/docker/daemon.json` (JSON puro: sin comentarios):

```json
{
  "registry-mirrors": ["https://dockerhub-proxy.lab.local"]
}
```

### Alternativas

| Producto | Notas |
|---|---|
| **Sonatype Nexus** | multi-formato; puerto 8081 por defecto |
| **JFrog Artifactory** | multi-formato; muy extendido en empresa |
| **Harbor** | especializado en imágenes y charts; escaneo de vulnerabilidades y firmas |
| **GitLab / GitHub Packages** | integrado con el repositorio de código y la CI |

### Buenas prácticas

- **Versiones inmutables** en los repositorios de releases: la `1.4.2` nunca se sobrescribe.
- Repositorios separados para `snapshots` y `releases`.
- Tokens de usuario o de servicio, no la contraseña personal, en CI.
- Políticas de limpieza (*cleanup policies*): los proxies y snapshots crecen sin límite.
- No publiques en un hosted interno con el **mismo nombre** que un paquete público: un group
  mal ordenado puede servir el público (*dependency confusion*).
