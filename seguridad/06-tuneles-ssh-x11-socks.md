# SSH — Túneles: X11 forwarding, port forwarding y SOCKS proxy

## X11

Protocolo que permite a los sistemas operativos Unix/Linux mostrar interfaces gráficas.

En el `~/.ssh/config` del cliente se añade:

```
ForwardX11 yes
ForwardX11Trusted yes
Compression yes
```

O directamente desde el terminal:

```bash
ssh -X usuario@192.168.1.50
```

## Xming

Windows no entiende X11 de forma nativa. Aquí entra **Xming**:

- Software ligero que instalas en Windows.
- Se ejecuta en segundo plano.
- Cuando una app Linux envía instrucciones gráficas a la red, Xming las recibe y las dibuja.

## Túnel SSH (X11 forwarding)

El protocolo X11 no es seguro por sí solo; por eso se encapsula en SSH.

Cuando haces `ssh -X usuario@servidor` (o activas la casilla *Enable X11 forwarding* en PuTTY):

- Se crea una conexión segura y cifrada entre tu PC y el servidor.
- SSH "engaña" a la app de Linux haciéndole creer que el servidor gráfico está ahí mismo.
- Toda la información gráfica se mete dentro del túnel SSH cifrado, viaja de forma segura,
  llega a Windows y SSH se la entrega a Xming.

## Local port forwarding

```bash
ssh -i ~/.ssh/mi_clave.pem -L 8080:localhost:80 usuario@servidor1.ejemplo.com
```

| Parte | Significado |
|---|---|
| `-i` | *identity*: ruta de la clave privada |
| `-L 8080` | puerto local que abres en tu PC |
| `localhost:80` | destino visto **desde el servidor** |
| `usuario@servidor1…` | usuario y servidor destino |

Resultado: `http://localhost:8080` en tu PC llega al puerto 80 del servidor remoto.

## SOCKS proxy (dynamic forwarding)

Sirve para pasar todo tu tráfico de internet por el servidor remoto.

Si ejecutas SSH con el parámetro `-D` (por ejemplo `ssh -D 8080 usuario@servidor`):

- Tu PC abre un puerto local (el 8080).
- Ese puerto se convierte en un **proxy SOCKS**.
- Si configuras tu navegador para que use ese proxy, todo lo que navegues parecerá que viene
  desde el servidor remoto. Es como crear tu propia VPN casera.

```bash
ssh -D -f -N 9090 servidor1.ejemplo.com     # luego configurar el navegador
```

| Flag | Significado |
|---|---|
| `-D` | SOCKS proxy |
| `-f` | SSH a segundo plano |
| `-N` | no ejecuta comandos: no sale nada en la terminal |

## Resumen de flags de SSH

| Flag | Significado |
|---|---|
| `-i` | *identity*: ruta de la clave privada |
| `-L` | local forwarding (traer un puerto remoto a tu PC) |
| `-R` | remote forwarding (llevar un puerto tuyo al servidor) |
| `-D` | dynamic forwarding (SOCKS proxy) |
| `-X` | X11 forwarding (apps gráficas de Linux) |
| `-p` | port (cambiar puerto) |
| `-N` | no command (no abre terminal, solo el túnel) |
| `-f` | fork background (SSH en segundo plano) |

## Los tres tipos de forwarding, en un diagrama

```
LOCAL  (-L)   "trae un puerto remoto a mi PC"
   tu PC :8080 ══════ túnel ══════► servidor ──► destino:80
   navegas a localhost:8080 y ves el servicio remoto

REMOTE (-R)   "lleva un puerto mío al servidor"
   servidor :9000 ◄═════ túnel ═════ tu PC ──► localhost:3000
   alguien en el servidor accede a tu app local

DYNAMIC (-D)  "usa el servidor como proxy para todo"
   tu PC :1080 (SOCKS) ══ túnel ══► servidor ──► cualquier destino
   el navegador sale a internet desde el servidor
```

## Casos de uso reales

| Situación | Comando |
|---|---|
| Acceder a una BBDD que solo escucha en localhost del servidor | `ssh -L 5432:localhost:5432 usuario@host` |
| Ver un panel interno (Grafana) sin exponerlo | `ssh -L 3000:grafana.interno:3000 usuario@bastion` |
| Alcanzar un equipo detrás del servidor | `ssh -L 8080:10.0.0.50:80 usuario@bastion` |
| Enseñar tu app local a alguien del servidor | `ssh -R 9000:localhost:3000 usuario@host` |
| Navegar como si estuvieras en la red del servidor | `ssh -D 1080 usuario@host` |

> Para `-R` con acceso desde otras máquinas hace falta `GatewayPorts yes` en el servidor;
> por defecto solo escucha en su localhost.

## Verificar y cerrar túneles

```bash
ss -tlnp | grep 8080            # ¿está abierto el puerto local?
ps aux | grep "ssh -"           # túneles en background (-f)
pkill -f "ssh -D 9090"          # cerrar uno concreto
```

Con sesión interactiva, la secuencia de escape de SSH:

```
   Enter  ~C     → abre la línea de comandos de ssh (añadir túneles en caliente)
   Enter  ~.     → cortar la conexión colgada
   Enter  ~?     → ayuda
```

## Túneles persistentes

Para que un túnel sobreviva a cortes, `autossh`:

```bash
autossh -M 0 -f -N -L 5432:localhost:5432 usuario@host \
  -o "ServerAliveInterval 30" -o "ServerAliveCountMax 3"
```

O como servicio de systemd:

```ini
[Unit]
Description=Tunel SSH a la BBDD
After=network-online.target

[Service]
ExecStart=/usr/bin/ssh -N -L 5432:localhost:5432 usuario@host
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

## Implicaciones de seguridad

```
   Un túnel SSH atraviesa el firewall "legítimamente":
   el tráfico va cifrado dentro del puerto 22.
   · -R y -D pueden usarse para sacar datos o dar acceso inverso a una red.
   · Por eso en entornos regulados se desactiva en el servidor:
       AllowTcpForwarding no
       PermitOpen 10.0.0.5:5432      (lista blanca)
       GatewayPorts no
       X11Forwarding no
```

Úsalos siempre en infraestructura propia o con autorización.

## X11: alternativas modernas

| Opción | Cuándo |
|---|---|
| `ssh -X` + Xming/VcXsrv | una app gráfica suelta en Linux |
| WSLg (Windows 11) | apps gráficas Linux en Windows sin servidor X aparte |
| RDP / xrdp | escritorio completo, mucho mejor rendimiento |
| VNC | escritorio persistente que sigue vivo al desconectar |

> `-X` aplica restricciones de seguridad X11; `-Y` (trusted) las relaja: no uses `-Y` con
> servidores en los que no confíes plenamente.

## Los tres tipos de forwarding, de un vistazo

```
 LOCAL  (-L 8080:destino:80)
   tu PC:8080 ──túnel──► servidor ──► destino:80
   "traigo un puerto remoto a mi máquina"
   uso típico: acceder a una BBDD o panel interno que no está expuesto

 REMOTE (-R 9000:localhost:3000)
   servidor:9000 ──túnel──► tu PC:3000
   "expongo un puerto mío en el servidor"
   uso típico: que un compañero o un webhook llegue a tu entorno local

 DYNAMIC (-D 1080)
   tu PC:1080 (proxy SOCKS) ──túnel──► servidor ──► cualquier destino
   "uso el servidor como salida a internet"
   uso típico: navegar como si estuvieras en la red del servidor
```

## Ejemplos completos

```bash
# Acceder a un PostgreSQL que solo escucha en localhost del servidor
ssh -L 5432:localhost:5432 usuario@servidor
psql -h localhost -U postgres        # en tu PC

# Acceder a un panel interno a través de un bastión
ssh -L 8080:panel.interno.local:80 usuario@bastion

# Exponer tu app local en el servidor (requiere GatewayPorts en sshd)
ssh -R 9000:localhost:3000 usuario@servidor

# Proxy SOCKS en background, sin terminal
ssh -D 1080 -f -N usuario@servidor
curl --socks5 localhost:1080 https://intranet.interno.local
```

Para que un `-R` sea accesible desde otras máquinas y no solo desde el propio servidor:

```
# en sshd_config del servidor
GatewayPorts yes
```

## Túneles persistentes

```bash
# autossh reabre el túnel si se cae
autossh -M 0 -f -N -L 5432:localhost:5432 usuario@servidor
```

O como servicio de systemd:

```ini
# /etc/systemd/system/tunel-db.service
[Unit]
Description=Tunel SSH a la BBDD
After=network-online.target

[Service]
User=pedro
ExecStart=/usr/bin/ssh -N -o ServerAliveInterval=30 -L 5432:localhost:5432 usuario@servidor
Restart=always

[Install]
WantedBy=multi-user.target
```

## Multiplexado: conexiones instantáneas

Reutiliza una única conexión TCP para todas las sesiones al mismo host (acelera muchísimo
Ansible y los `scp` repetidos).

```
Host *
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
```

```
   1ª conexión: handshake completo (~300 ms)
   2ª y siguientes: reutilizan el socket (~5 ms)
```

En Ansible el equivalente es el *pipelining* + `ssh_args` con `ControlPersist`, configurado en
`ansible.cfg`.

## Seguridad de los túneles

| Riesgo | Mitigación |
|---|---|
| Un usuario abre túneles hacia la red interna | `AllowTcpForwarding no` para cuentas que no lo necesiten |
| `-R` expuesto a toda la red | dejar `GatewayPorts no` salvo necesidad |
| Túnel olvidado abierto durante días | usar `ControlPersist` corto y auditar con `ss -tlnp` |
| X11 forwarding da acceso a tu escritorio | `X11Forwarding no` por defecto; usar `-X` solo cuando haga falta |

```bash
ss -tlnp | grep ssh        # ver qué puertos locales ha abierto tu ssh
```
