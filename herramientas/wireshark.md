# Wireshark

Analizador de protocolos de red (sniffer o capturador de paquetes). Herramienta estándar de la
industria para saber qué pasa dentro de una red informática.

## Qué hace

- **Captura datos en tiempo real**: se conecta a la tarjeta de red y "escucha" todo el tráfico.
- **Disección de protocolos**: traduce el binario a forma visual y legible.
- **Filtrado avanzado**: por ejemplo, "solo tráfico de 192.168.1.50 y certificados TLS".

## La ventana principal

```
┌───────────────────────────────────────────────────────────────────────────┐
│ [ filtro de visualización: tcp.port == 443 && ip.addr == 10.0.0.15     ]  │
├───────────────────────────────────────────────────────────────────────────┤
│ No.  Time    Source       Destination  Protocol  Info                     │  ← 1. lista de
│ 1    0.000   10.0.0.20    10.0.0.15    TCP       51544 → 443 [SYN]        │     paquetes
│ 2    0.001   10.0.0.15    10.0.0.20    TCP       443 → 51544 [SYN, ACK]   │
│ 3    0.001   10.0.0.20    10.0.0.15    TLSv1.3   Client Hello             │
├───────────────────────────────────────────────────────────────────────────┤
│ ▸ Ethernet II                                                             │  ← 2. detalle por
│ ▸ Internet Protocol Version 4, Src: 10.0.0.20, Dst: 10.0.0.15             │     capas (clic
│ ▾ Transmission Control Protocol, Src Port: 51544, Dst Port: 443           │     derecho → usar
│     Flags: 0x002 (SYN)                                                    │     como filtro)
├───────────────────────────────────────────────────────────────────────────┤
│ 0000  00 1a 2b 3c 4d 5e 00 0c  29 aa bb cc 08 00 45 00   ..+<M^..).....E. │  ← 3. bytes en
└───────────────────────────────────────────────────────────────────────────┘     hexadecimal
```

## Dos tipos de filtro (y no se escriben igual)

| | Filtro de **captura** | Filtro de **visualización** |
|---|---|---|
| Cuándo actúa | **antes** de capturar: lo que no cumple, no se guarda | **después**: oculta, pero sigue en la captura |
| Sintaxis | BPF (la de `tcpdump`) | la propia de Wireshark |
| Ejemplo | `host 10.0.0.15 and port 443` | `ip.addr == 10.0.0.15 && tcp.port == 443` |
| Para qué | capturas largas o en servidores con mucho tráfico | analizar |

### Filtros de captura (BPF)

```
host 10.0.0.15
net 192.168.1.0/24
port 53
tcp port 443
not port 22                      ← imprescindible si capturas por SSH: si no, capturas tu propia sesión
host 10.0.0.15 and (port 80 or port 443)
```

## Ejemplos de filtros

```
ip.addr == 192.168.1.50
tcp.port == 443
tls.handshake.type == 1
http.request.method == "POST"
```

### Chuleta de filtros de visualización

| Filtro | Muestra |
|---|---|
| `ip.src == 10.0.0.20` / `ip.dst == …` | origen / destino |
| `ip.addr == 192.168.1.0/24` | toda una red |
| `tcp.port in {80 443 8080}` | varios puertos |
| `tcp.flags.syn == 1 && tcp.flags.ack == 0` | intentos de conexión (solo SYN) |
| `tcp.flags.reset == 1` | conexiones rechazadas o cortadas |
| `tcp.analysis.retransmission` | retransmisiones (pérdida de paquetes) |
| `tcp.analysis.zero_window` | el receptor está saturado |
| `tcp.stream eq 5` | una conversación TCP concreta |
| `dns` / `dns.qry.name contains "lab.local"` | tráfico DNS / consultas por nombre |
| `dns.flags.rcode != 0` | respuestas DNS con error (NXDOMAIN = 3) |
| `http.response.code >= 400` | errores HTTP (solo HTTP sin cifrar) |
| `tls.handshake.extensions_server_name == "api.lab.local"` | conexiones TLS a ese nombre (SNI) |
| `tls.handshake.type == 11` | certificado del servidor (visible solo hasta TLS 1.2) |
| `frame contains "password"` | cualquier paquete que contenga ese texto |
| `!(arp \|\| dns \|\| icmp)` | quitar ruido |
| `kerberos` / `ldap` / `smb2` | tráfico de Active Directory y carpetas compartidas |

| Operador | Significado |
|---|---|
| `==` `!=` `>` `<` | comparación |
| `&&` `\|\|` `!` | y, o, no (también `and`, `or`, `not`) |
| `contains` | contiene el texto |
| `matches` | expresión regular (`http.host matches "^web[0-9]+"`) |
| `in {…}` | pertenece al conjunto |

> Truco: en el panel de detalle, **clic derecho sobre cualquier campo → Aplicar como filtro**.
> Así se aprenden los nombres de los campos sin memorizarlos.

## Leer un handshake TLS en la captura

```
   CLIENTE                                              SERVIDOR
     │── TCP SYN / SYN-ACK / ACK ──────────────────────────│   ← la conexión TCP
     │── Client Hello  (type 1)  ─────────────────────────►│   SNI, versiones y cifrados que ofrece
     │◄─ Server Hello  (type 2)  ──────────────────────────│   versión y cifrado elegidos
     │◄─ Certificate   (type 11, solo hasta TLS 1.2) ──────│   en TLS 1.3 el certificado ya va cifrado
     │══ Application Data (cifrado) ═══════════════════════│
```

Relación con la teoría: [TLS, SSL y PKI](../seguridad/07-tls-ssl-pki.md).

### Descifrar tráfico TLS propio

Si controlas el cliente, puedes pedirle que vuelque las claves de sesión:

```bash
export SSLKEYLOGFILE=$HOME/tls-keys.log
curl https://app.lab.local/api/health          # curl, Chrome y Firefox respetan esta variable
```

En Wireshark: *Preferencias → Protocols → TLS → (Pre)-Master-Secret log filename* → ese
fichero. El tráfico HTTPS aparece como HTTP legible.

> Ese fichero permite descifrar las sesiones capturadas: trátalo como un secreto y bórralo al
> terminar.

## Capturar en servidores: tcpdump y tshark

> Para capturar en un servidor sin interfaz gráfica se usa `tshark` (la versión CLI) o
> `tcpdump`, y luego se abre el `.pcap` en Wireshark.

```bash
# capturar a fichero
sudo tcpdump -i eth0 -nn -w /tmp/captura.pcap 'host 10.0.0.15 and port 5432'

# ver en pantalla, sin resolver nombres, 50 paquetes
sudo tcpdump -i any -nn -c 50 'port 53'

# capturas largas: ficheros rotatorios de 100 MB, máximo 5
sudo tcpdump -i eth0 -nn -w /tmp/cap.pcap -C 100 -W 5 'not port 22'
```

| Opción de tcpdump | Qué hace |
|---|---|
| `-i eth0` / `-i any` | interfaz / todas |
| `-nn` | sin resolver nombres ni puertos (más rápido y claro) |
| `-w fichero.pcap` | guardar para Wireshark |
| `-c 100` | parar tras 100 paquetes |
| `-A` / `-X` | mostrar contenido en ASCII / hexadecimal |
| `-C 100 -W 5` | rotar ficheros |

```bash
# tshark: los filtros de visualización de Wireshark, en consola
tshark -i eth0 -f "port 53" -Y "dns.flags.rcode != 0"
tshark -r captura.pcap -Y http.request -T fields -e ip.src -e http.host -e http.request.uri
```

### Captura remota en directo

```bash
ssh root@srv01 "tcpdump -i eth0 -U -w - 'not port 22'" | wireshark -k -i -
```

La captura se hace en el servidor y los paquetes llegan en vivo al Wireshark de tu equipo.

### En Kubernetes

```bash
# contenedor efímero con herramientas de red dentro del pod (comparte su red)
kubectl debug -it pod/mi-app --image=nicolaka/netshoot --target=app -- \
  tcpdump -i any -nn port 8080
```

## Menús que más se usan

| Menú | Para qué |
|---|---|
| Clic derecho → **Follow → TCP Stream** | ver la conversación completa reconstruida |
| **Statistics → Conversations** | quién habla con quién y cuánto |
| **Statistics → Protocol Hierarchy** | reparto del tráfico por protocolo |
| **Statistics → I/O Graphs** | tráfico en el tiempo (picos, cortes) |
| **Analyze → Expert Information** | resumen de problemas detectados: retransmisiones, resets, errores |

## Patrones de diagnóstico

| Ves | Significa |
|---|---|
| `SYN` sin respuesta, repetido | firewall descartando: puerto **filtrado** |
| `SYN` → `RST, ACK` | host vivo, **nadie escucha** en ese puerto |
| Muchas `Retransmission` / `Dup ACK` | pérdida de paquetes en la red |
| `TCP ZeroWindow` | el receptor no da abasto (aplicación lenta leyendo) |
| Client Hello → `Alert` o `RST` | TLS incompatible: versión, cifrados o SNI no reconocido |
| DNS con `No such name` | el nombre no existe en ese resolver |
| Mucho tiempo entre la petición y la respuesta HTTP con la red fluida | el problema está en la **aplicación**, no en la red |

## Instalación y permisos

- **Windows**: el instalador incluye **Npcap**, el driver de captura.
- **Debian/Ubuntu**: capturar sin ser root:

```bash
sudo apt install wireshark
sudo dpkg-reconfigure wireshark-common      # permitir captura a usuarios no root → Sí
sudo usermod -aG wireshark $USER            # y volver a iniciar sesión
```

## Uso responsable

- Capturar solo en redes y equipos propios o con autorización: el tráfico ajeno puede
  contener datos personales y credenciales.
- Un `.pcap` puede incluir contraseñas, cookies y tokens: no lo adjuntes a un ticket sin
  filtrarlo antes.
