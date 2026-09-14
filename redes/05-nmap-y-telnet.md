# Redes — Nmap y Telnet

## Nmap (Network Mapper)

Herramienta de código abierto y gratuita que se utiliza para la exploración de redes y la
auditoría de seguridad. Funciona enviando paquetes de red modificados y esperando su
respuesta.

### Cuatro tareas principales

**1) Descubrimiento de hosts (host discovery)**
Averigua qué dispositivos están encendidos y conectados a la red. Evita escanear IPs
inactivas.

**2) Escaneo de puertos (port scanning)**
Escanea los 65.535 puertos lógicos de una máquina. Cada puerto puede estar:
`open`, `closed` o `filtered`.

**3) Detección de versiones y servicios**
No solo dice que el puerto está abierto, sino que lo interroga y sabe el software exacto que
ejecuta (p. ej. `OpenSSH 9.2p1`).

**4) Detección del sistema operativo (OS detection)**
Con *fingerprinting* puede adivinar con un alto porcentaje de acierto qué sistema operativo
utiliza el objetivo (Linux, Windows, macOS) y qué versión de kernel tiene.

### Cómo decide el estado de un puerto (SYN scan)

```
   NMAP                                  OBJETIVO
     │──── SYN ─────────────────────────────►│
     │                                        │
     │◄─── SYN-ACK ───────────────────────────│   → open      (nmap responde RST
     │                                        │                y no completa la conexión)
     │◄─── RST ───────────────────────────────│   → closed    (host vivo, nadie escucha)
     │                                        │
     │     (nada) / ICMP unreachable          │   → filtered  (un firewall en medio)
```

| Estado | Significado |
|---|---|
| `open` | hay un servicio aceptando conexiones |
| `closed` | el host responde, pero no hay nada escuchando |
| `filtered` | no hay respuesta o llega un ICMP de rechazo: firewall |
| `unfiltered` | responde, pero no se puede saber si está abierto (escaneo ACK) |
| `open\|filtered` | típico de UDP: el silencio puede ser "abierto" o "filtrado" |

### Tipos de escaneo

| Opción | Qué hace | Notas |
|---|---|---|
| `-sn` | solo descubrimiento de hosts, sin puertos | "¿quién está vivo en esta red?" |
| `-sS` | SYN scan (medio abierto) | por defecto con root; rápido y discreto |
| `-sT` | connect scan (handshake completo) | por defecto sin root; queda en los logs del servicio |
| `-sU` | escaneo UDP | lento; combinar con `--top-ports` |
| `-sV` | detección de versiones | |
| `-O` | detección de sistema operativo | requiere root |
| `-A` | `-sV` + `-O` + scripts por defecto + traceroute | ruidoso |
| `-Pn` | no hacer ping previo: da el host por vivo | para hosts que bloquean ICMP (Windows) |

### Opciones de puertos, velocidad y salida

| Opción | Qué hace |
|---|---|
| `-p 22,80,443` | puertos concretos |
| `-p 1-1024` / `-p-` | rango / los 65.535 |
| `--top-ports 100` | los 100 más frecuentes |
| `--open` | muestra solo los abiertos |
| `-T0` … `-T5` | plantillas de velocidad (`-T4` es habitual en redes propias) |
| `-n` | sin resolución DNS inversa (más rápido) |
| `-oN fichero.txt` / `-oX fichero.xml` / `-oG` | salida normal / XML / "grepeable" |
| `-oA base` | las tres a la vez |
| `-iL hosts.txt` | leer objetivos de un fichero |

### Recetas

```bash
# ¿qué equipos hay encendidos en la red del lab?
nmap -sn 192.168.1.0/24

# puertos abiertos y versiones de un servidor
sudo nmap -sS -sV --open -p- -T4 192.168.1.50

# auditar la configuración TLS de un servicio
nmap --script ssl-enum-ciphers -p 443 app.lab.local

# ver el certificado que presenta
nmap --script ssl-cert -p 443 app.lab.local

# servicios UDP más comunes
sudo nmap -sU --top-ports 20 192.168.1.1

# ¿los nodos de Kubernetes exponen algo que no deberían?
nmap -Pn -p 2379,2380,6443,10250,30000-32767 --open 192.168.1.60-62
```

### NSE (Nmap Scripting Engine)

Permite automatizar tareas avanzadas mediante scripts en lenguaje **Lua**: buscar
vulnerabilidades, detectar infecciones (malware o backdoors) y realizar pruebas avanzadas de
autenticación.

| Categoría | Ejemplos |
|---|---|
| `default` (`-sC`) | scripts seguros que se lanzan con `-A` |
| `discovery` | `http-title`, `smb-os-discovery`, `dns-brute` |
| `safe` | no deberían tumbar ni alterar nada |
| `vuln` | comprobación de vulnerabilidades conocidas |
| `auth` / `brute` | autenticación y fuerza bruta (**ojo**: bloqueo de cuentas) |

```bash
ls /usr/share/nmap/scripts/ | grep http        # scripts disponibles
nmap --script-help ssl-enum-ciphers
nmap --script "safe and discovery" -p 80,443 192.168.1.50
```

> Nota de uso: escanear redes o equipos que no son tuyos ni tienes autorización para auditar
> puede ser ilegal. Usarlo solo en infraestructura propia o con permiso explícito.

Incluso en tu propia empresa, avisa antes: un escaneo dispara alertas del IDS o del SOC y
un `-T5` o los scripts `brute` pueden tumbar servicios frágiles o bloquear cuentas.

## Telnet

Protocolo de red antiguo que se utiliza para acceder y gestionar un ordenador de forma remota.
Fue el estándar absoluto durante décadas para administrar routers, switches y servidores.

> Va en texto plano, sin cifrado: hoy está sustituido por SSH. Su único uso razonable es como
> herramienta de diagnóstico rápida para comprobar si un puerto responde:
> `telnet host 443`.

### Leer el resultado

```
$ telnet 10.0.0.15 5432
Trying 10.0.0.15...
Connected to 10.0.0.15.          ← puerto abierto
Escape character is '^]'.

$ telnet 10.0.0.15 5433
Trying 10.0.0.15...
telnet: Unable to connect to remote host: Connection refused   ← cerrado

$ telnet 10.0.0.15 5434
Trying 10.0.0.15...              ← se queda aquí → filtrado (firewall)
```

Salir: `Ctrl + ]` y después `quit`.

### Hablar el protocolo a mano

Con protocolos de texto se puede "conversar" con el servicio:

```
$ telnet web.lab.local 80
GET / HTTP/1.1
Host: web.lab.local
                                  ← línea en blanco para enviar
HTTP/1.1 200 OK
...
```

```
$ telnet smtp.lab.local 25
220 smtp.lab.local ESMTP
EHLO prueba
250-smtp.lab.local
...
QUIT
```

### Instalarlo en Windows (viene desactivado)

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName TelnetClient
```

## Alternativas modernas para "¿responde este puerto?"

| Herramienta | Comando |
|---|---|
| netcat | `nc -zv 10.0.0.15 5432` |
| PowerShell | `Test-NetConnection 10.0.0.15 -Port 5432` |
| bash puro (sin instalar nada) | `timeout 3 bash -c '</dev/tcp/10.0.0.15/5432' && echo abierto` |
| curl | `curl -v telnet://10.0.0.15:5432` |
| OpenSSL (puertos TLS) | `openssl s_client -connect app.lab.local:443 -servername app.lab.local` |

> El truco de `/dev/tcp` es oro dentro de contenedores mínimos donde no hay ni `nc` ni
> `telnet`, pero sí `bash`.
