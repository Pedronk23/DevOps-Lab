# Redes — Puertos y protocolos

## Dónde vive cada cosa: modelo TCP/IP

```
   Capa              Qué identifica        Ejemplos
   ────────────────  ────────────────────  ─────────────────────────────
   Aplicación        el protocolo "útil"   HTTP, SSH, DNS, SMTP, TLS
   Transporte        el PUERTO             TCP, UDP
   Red (Internet)    la IP                 IPv4, IPv6, ICMP
   Enlace            la MAC                Ethernet, Wi-Fi, ARP
```

| OSI (7 capas) | TCP/IP (4 capas) |
|---|---|
| 7 Aplicación · 6 Presentación · 5 Sesión | Aplicación |
| 4 Transporte | Transporte |
| 3 Red | Internet |
| 2 Enlace · 1 Física | Acceso a red |

> Cuando alguien dice "balanceador L4" habla de puertos (TCP/UDP); "L7" es que entiende HTTP
> (rutas, cabeceras, hosts). Un Service de Kubernetes es L4; un Ingress es L7.

## Puertos

- Una IP tiene **2^16 = 65.536 puertos**.
- La **IP identifica el dispositivo**; el **puerto identifica el servicio o aplicación**
  dentro de ese dispositivo.

### Rangos importantes

| Rango | Uso |
|---|---|
| 0 – 1023 | puertos bien conocidos (*well-known*) |
| 1024 – 49151 | aplicaciones y servicios (registrados) |
| 49152 – 65535 | puertos temporales / dinámicos (efímeros) |

> **IANA** (*Internet Assigned Numbers Authority*): entidad principal que asigna los puertos.

Cada IP tiene esos puertos para los dos protocolos principales: **TCP** y **UDP**.

- En Linux, abrir un puerto **< 1024** requiere root (o la *capability* `CAP_NET_BIND_SERVICE`).
  Por eso muchos contenedores escuchan en 8080 en vez de 80.
- El rango efímero real depende del sistema operativo:

```bash
cat /proc/sys/net/ipv4/ip_local_port_range     # Linux: 32768 60999 por defecto
netsh int ipv4 show dynamicport tcp            # Windows: 49152 - 65535
```

### Socket: la conexión completa

Una conexión TCP se identifica por **5 valores** (*5-tuple*):

```
   protocolo   IP origen      puerto origen   IP destino    puerto destino
   TCP         192.168.1.10   51544 (efímero) 10.0.0.15     443
```

Por eso un servidor puede atender miles de clientes en el mismo puerto 443: cada conexión se
distingue por la IP y el puerto de origen del cliente.

## TCP (Transmission Control Protocol)

- Prioriza que los datos **lleguen correctamente**.
- Verifica que lleguen los paquetes, que lleguen en orden y sin errores.
  Si algo falla, lo vuelve a enviar.
- Características: conexión estable, más lento, confirma recepción, control de errores.
- Usos típicos: web (HTTP/HTTPS), emails, descargas, SSH, bases de datos.

### Three-way handshake (abrir conexión)

```
   CLIENTE                                   SERVIDOR (LISTEN :443)
      │──── SYN  (seq=x) ──────────────────────►│
      │◄─── SYN-ACK (seq=y, ack=x+1) ───────────│
      │──── ACK  (ack=y+1) ────────────────────►│
      │════ ESTABLISHED: ya se pueden mandar datos ═│
```

### Cierre

```
      │──── FIN ───────────────────────────────►│
      │◄─── ACK ────────────────────────────────│
      │◄─── FIN ────────────────────────────────│
      │──── ACK ───────────────────────────────►│
   TIME_WAIT (espera ~2 min antes de liberar el puerto)
```

### Flags TCP

| Flag | Significado |
|---|---|
| `SYN` | quiero abrir conexión |
| `ACK` | confirmo lo recibido |
| `FIN` | he terminado de enviar |
| `RST` | corta en seco: "aquí no hay nadie" o conexión abortada |
| `PSH` | entrega los datos a la aplicación ya |

> Un `RST` como respuesta a un `SYN` significa **puerto cerrado** (el host existe pero nadie
> escucha). **Sin respuesta** suele significar **firewall** descartando paquetes.

### Estados que verás en `ss` / `netstat`

| Estado | Qué indica |
|---|---|
| `LISTEN` | un proceso espera conexiones en ese puerto |
| `ESTABLISHED` | conexión activa |
| `TIME_WAIT` | cerrada por nuestro lado, esperando; muchos = mucho tráfico de conexiones cortas |
| `CLOSE_WAIT` | el otro cerró y **nuestra aplicación no** ha cerrado el socket; muchos = bug de la app |
| `SYN_SENT` | enviamos SYN y no llega respuesta (firewall o host caído) |

## UDP (User Datagram Protocol)

- Prioriza la **velocidad**.
- No confirma si llegaron, si llegaron en orden ni si se perdieron.
- Características: muy rápido, menor latencia, sin confirmaciones, puede perder paquetes.
- Usos típicos: videojuegos online, streaming, videollamadas, DNS.

## TCP vs UDP

| | TCP | UDP |
|---|---|---|
| Conexión | sí (handshake) | no |
| Orden garantizado | sí | no |
| Retransmisión | sí | no (si hace falta, la hace la aplicación) |
| Cabecera | 20+ bytes | 8 bytes |
| Latencia | mayor | menor |
| Ejemplos | HTTP/1.1, HTTP/2, SSH, SMTP, PostgreSQL | DNS, NTP, DHCP, SNMP, VXLAN, WireGuard, QUIC (HTTP/3) |

> **HTTP/3** va sobre **QUIC**, que va sobre **UDP**: implementa la fiabilidad por encima,
> en espacio de usuario. Si un firewall solo abre 443/TCP, HTTP/3 cae a HTTP/2.

**ICMP** no usa puertos: es el protocolo de `ping` y de los mensajes de error de red
("destino inalcanzable", "TTL excedido" en `traceroute`).

## Ejemplos para tenerlo claro

- **TCP**: al descargar un archivo no puede faltar ni un byte → TCP lo garantiza.
- **UDP**: en una videollamada, mejor perder un frame que esperar a retransmitirlo y
  generar retraso.

## Ver puertos y conexiones

```bash
# Linux
ss -tulpn                         # puertos en escucha TCP/UDP con el proceso
ss -tan state established         # conexiones TCP activas
ss -tan | awk '{print $1}' | sort | uniq -c    # recuento por estado
sudo lsof -i :8080                # qué proceso usa el 8080

# probar si un puerto remoto responde
nc -zv 10.0.0.15 443              # TCP
nc -zvu 10.0.0.53 53              # UDP (poco fiable: UDP no confirma)
```

```powershell
# Windows
netstat -ano | findstr :443                         # conexiones y PID
Get-NetTCPConnection -State Listen | Sort-Object LocalPort
Get-Process -Id (Get-NetTCPConnection -LocalPort 8080).OwningProcess
Test-NetConnection 10.0.0.15 -Port 443              # TcpTestSucceeded : True
```

## Diagnóstico rápido

| Resultado al conectar | Qué significa |
|---|---|
| `Connection refused` | llega al host pero nadie escucha en ese puerto (RST) |
| `Connection timed out` | no hay respuesta: firewall, ruta o host apagado |
| `No route to host` | problema de enrutamiento o un firewall que responde con ICMP |
| Conecta pero no responde | el servicio acepta pero está colgado, o hay un proxy/balanceador en medio |
| El servicio escucha en `127.0.0.1:8080` | solo acepta conexiones **locales**: debe escuchar en `0.0.0.0` |
