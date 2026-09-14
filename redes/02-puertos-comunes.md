# Redes — Puertos más comunes

| Puerto | Protocolo | Uso |
|---|---|---|
| 20 / 21 | TCP | FTP (transferencia de archivos) |
| 22 | TCP | SSH (acceso remoto seguro) |
| 23 | TCP | Telnet |
| 25 | TCP | SMTP (correo saliente) |
| 53 | TCP/UDP | DNS |
| 67 / 68 | UDP | DHCP |
| 80 | TCP | HTTP |
| 110 | TCP | POP3 |
| 123 | UDP | NTP (hora) |
| 143 | TCP | IMAP |
| 161 | UDP | SNMP |
| 443 | TCP | HTTPS |
| 3306 | TCP | MySQL |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 8080 | TCP | HTTP alternativo |
| 27017 | TCP | MongoDB |

> DNS usa **UDP** para las consultas normales y **TCP** cuando la respuesta es grande
> (DNSSEC, transferencias de zona). Si abres solo 53/UDP, algunas resoluciones fallarán.

## Por categorías

### Web, correo y variantes cifradas

| Puerto | Protocolo | Uso |
|---|---|---|
| 443 | UDP | HTTP/3 (QUIC) |
| 8443 | TCP | HTTPS alternativo (paneles de administración) |
| 465 | TCP | SMTPS (SMTP con TLS implícito) |
| 587 | TCP | SMTP *submission* (clientes que envían correo, con STARTTLS) |
| 993 | TCP | IMAPS |
| 995 | TCP | POP3S |
| 853 | TCP | DNS over TLS |
| 990 | TCP | FTPS implícito |

### Windows y directorio activo

| Puerto | Protocolo | Uso |
|---|---|---|
| 88 | TCP/UDP | Kerberos |
| 135 | TCP | RPC Endpoint Mapper |
| 137-139 | UDP/TCP | NetBIOS (legado) |
| 389 | TCP/UDP | LDAP |
| 636 | TCP | LDAPS |
| 445 | TCP | SMB (carpetas compartidas, `\\servidor\C$`) |
| 3268 / 3269 | TCP | Catálogo global de AD (LDAP / LDAPS) |
| 3389 | TCP/UDP | RDP |
| 5985 | TCP | WinRM HTTP |
| 5986 | TCP | WinRM HTTPS |
| 1433 | TCP | Microsoft SQL Server |

### Kubernetes

| Puerto | Protocolo | Uso |
|---|---|---|
| 6443 | TCP | API Server |
| 2379 / 2380 | TCP | etcd (clientes / entre miembros) |
| 10250 | TCP | kubelet API |
| 10257 | TCP | kube-controller-manager |
| 10259 | TCP | kube-scheduler |
| 30000-32767 | TCP/UDP | rango de NodePort |
| 179 | TCP | BGP (Calico, MetalLB en modo BGP) |
| 7946 | TCP/UDP | memberlist de MetalLB (modo L2) |
| 4789 | UDP | VXLAN (Calico y otros CNI) |
| 8472 | UDP | VXLAN de Flannel |

### Contenedores, secretos y artefactos

| Puerto | Protocolo | Uso |
|---|---|---|
| 2375 | TCP | Docker daemon **sin TLS** (no exponer nunca) |
| 2376 | TCP | Docker daemon con TLS |
| 5000 | TCP | Docker Registry |
| 8081 | TCP | Nexus Repository (por defecto) |
| 8200 / 8201 | TCP | Vault / OpenBao (API / clúster) |

### Observabilidad

| Puerto | Protocolo | Uso |
|---|---|---|
| 9090 | TCP | Prometheus |
| 9093 | TCP | Alertmanager |
| 9100 | TCP | Node Exporter |
| 9182 | TCP | windows_exporter |
| 3000 | TCP | Grafana |
| 3100 | TCP | Loki |
| 9200 | TCP | Elasticsearch / OpenSearch |
| 514 | UDP/TCP | Syslog |
| 4317 / 4318 | TCP | OpenTelemetry (gRPC / HTTP) |

### Mensajería y caché

| Puerto | Protocolo | Uso |
|---|---|---|
| 5672 | TCP | RabbitMQ (AMQP) |
| 15672 | TCP | RabbitMQ (panel web) |
| 9092 | TCP | Kafka |
| 11211 | TCP | Memcached |
| 1521 | TCP | Oracle Database |

### VPN

| Puerto | Protocolo | Uso |
|---|---|---|
| 1194 | UDP | OpenVPN |
| 51820 | UDP | WireGuard |
| 500 / 4500 | UDP | IPsec (IKE / NAT-T) |

## Consultar qué es un puerto

```bash
getent services 5432              # postgresql  5432/tcp
grep -w 9100 /etc/services        # no todo está: los puertos "modernos" no suelen salir
```

En Windows la misma tabla está en `C:\Windows\System32\drivers\etc\services`.

> Estos son los puertos **por defecto**. Nada impide que un servicio escuche en otro, y
> cambiar el puerto no es una medida de seguridad seria: `nmap -sV` lo identifica igual.

## Puertos que se recomienda cerrar (o no exponer a Internet)

- **Acceso remoto**: 22 (SSH, acceso remoto Linux), 23 (Telnet, obsoleto),
  3389 (RDP, acceso remoto Windows).
- **Servicios de red internos**: 135, 137-139, 445 → compartición de archivos en red.
- **Servicios antiguos de correo**: 25 (SMTP), 110 (POP3), 143 (IMAP).
- **Bases de datos**: 3306 (MySQL), 5432 (PostgreSQL), 6379 (Redis), 27017 (MongoDB).

Y además, por lo que dan si quedan expuestos:

| Puerto | Riesgo |
|---|---|
| 2375 | Docker sin TLS: equivale a dar root en el host |
| 6443, 10250 | API de Kubernetes y kubelet: control del clúster |
| 2379 | etcd: contiene **todos** los Secrets del clúster |
| 5985 / 5986 | WinRM: ejecución remota en Windows |
| 9200, 11211 | Elasticsearch y Memcached: históricamente expuestos sin autenticación |
| 161 | SNMP v1/v2c con comunidad `public`: filtra la configuración del equipo |

```
   Internet ──► [ firewall ] ──► 443 (Ingress / proxy inverso)
                    │
                    ├── 22, 3389  → solo por VPN o bastión
                    └── BBDD, etcd, WinRM, Docker → nunca desde fuera
```

## Abrir o comprobar en el firewall

```bash
# Debian/Ubuntu (ufw)
sudo ufw allow 443/tcp
sudo ufw status numbered

# RHEL/Rocky (firewalld)
sudo firewall-cmd --permanent --add-port=9100/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

```powershell
# Windows
New-NetFirewallRule -DisplayName "Node exporter" -Direction Inbound `
  -Protocol TCP -LocalPort 9182 -Action Allow
Get-NetFirewallRule -Enabled True -Direction Inbound | Select-Object DisplayName
```
