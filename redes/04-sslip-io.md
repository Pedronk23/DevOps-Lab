# Redes — sslip.io

Servicio DNS público con un truco: **la IP va escrita dentro del propio nombre**, y el
servicio simplemente la "lee" y la devuelve. No hay que registrar nada.

```
192-168-1-50.sslip.io  →  (consulta sslip.io)  →  192.168.1.50
```

La IP está literalmente en el nombre, pero con guiones en vez de puntos. sslip.io lee esa
parte, la convierte a la IP y la responde.

## Cómo resuelve

```
   tu PC                resolver DNS            servidores DNS de sslip.io
     │ grafana.192-168-1-240.sslip.io ?  │                  │
     │──────────────────►│                                  │
     │                   │── ¿quién lleva sslip.io? ───────►│
     │                   │◄── A 192.168.1.240 ──────────────│  (la saca del nombre)
     │◄── 192.168.1.240 ─│
     │
     └──► conecta a 192.168.1.240  (tu red local; sslip.io no ve ese tráfico)
```

sslip.io **solo responde al DNS**. El tráfico HTTP va directo a la IP, que puede ser privada.

## Formatos aceptados

| Nombre | Resuelve a |
|---|---|
| `192-168-1-50.sslip.io` | `192.168.1.50` |
| `192.168.1.50.sslip.io` | `192.168.1.50` (también con puntos) |
| `app.192-168-1-50.sslip.io` | `192.168.1.50` (subdominio delante) |
| `app-192-168-1-50.sslip.io` | `192.168.1.50` (prefijo con guion) |
| `--1.sslip.io` | `::1` (IPv6: los `:` se escriben como `-`) |

```bash
dig +short app.192-168-1-50.sslip.io        # 192.168.1.50
nslookup 10-0-0-15.sslip.io
Resolve-DnsName 10-0-0-15.sslip.io          # PowerShell
```

## Para qué sirve en la práctica

- Probar **Ingress** con nombres de host reales sin tocar DNS ni `/etc/hosts`.
- Tener un FQDN válido para certificados o pruebas de enrutamiento por host.
- Entornos de laboratorio y demos.

## Ejemplo: varias apps detrás de un Ingress en el lab

Con MetalLB asignando `192.168.1.240` al Ingress Controller (ver
[LoadBalancer y MetalLB](../kubernetes/06-networking-loadbalancer-metallb.md)):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: herramientas
spec:
  ingressClassName: nginx
  rules:
    - host: grafana.192-168-1-240.sslip.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grafana
                port:
                  number: 80
    - host: argocd.192-168-1-240.sslip.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
```

```
   grafana.192-168-1-240.sslip.io ─┐
                                   ├──► 192.168.1.240 (Ingress) ──► enruta por el Host
   argocd.192-168-1-240.sslip.io ──┘
```

Una sola IP, tantos nombres como apps, sin tocar ningún DNS.

## Certificados TLS

| Opción | ¿Funciona con IP privada? |
|---|---|
| Let's Encrypt **HTTP-01** | **no**: Let's Encrypt tiene que llegar a tu IP desde Internet |
| Let's Encrypt **DNS-01** | no: no controlas la zona `sslip.io` |
| CA interna con **cert-manager** (`ClusterIssuer` tipo CA) | **sí**: lo habitual en el lab |
| Certificado autofirmado | sí (con avisos en el navegador) |

Con una IP **pública**, HTTP-01 sí funciona, porque el nombre resuelve a una IP accesible.

## Limitaciones y trampas

- **Depende de Internet**: si el lab está aislado o sin DNS externo, no resuelve.
- **Protección DNS rebinding**: algunos routers y resolvers (dnsmasq con `stop-dns-rebind`,
  pfSense, OpenWrt, Pi-hole) **descartan respuestas públicas que apuntan a IPs privadas**.
  Síntoma: `dig @8.8.8.8` resuelve pero tu resolver local no. Solución: añadir `sslip.io` a
  la lista de excepciones del router.
- Es un servicio de terceros: **no para producción**.
- Cualquiera puede ver qué nombres consultas (el DNS no va cifrado).

## Alternativas

| Opción | Cuándo |
|---|---|
| `nip.io` | mismo concepto, otro proveedor |
| `/etc/hosts` o `C:\Windows\System32\drivers\etc\hosts` | una o dos entradas en tu máquina |
| dnsmasq con comodín | lab sin Internet: `address=/lab.local/192.168.1.240` resuelve `*.lab.local` |
| CoreDNS / DNS interno de la empresa | entornos compartidos y producción |
