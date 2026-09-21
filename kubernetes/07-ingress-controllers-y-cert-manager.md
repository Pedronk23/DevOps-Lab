# Kubernetes Networking (2) — Ingress Controllers (NGINX / Traefik) y cert-manager

En vez de usar 50 IPs públicas, lo cual es costoso, usas **una sola IP pública** que apunte a
un Ingress Controller.

El Ingress Controller es un software de tipo **proxy inverso** que actúa como la recepción
central del clúster. Escucha todo el tráfico entrante (normalmente por los puertos 80 y 443)
y, leyendo las cabeceras HTTP, decide a qué app interna enviar la petición.

## NGINX Ingress Controller

Es el estándar en la industria, mantenido por el proyecto Kubernetes y basado en el motor
clásico del servidor web NGINX.

- **Arquitectura**: funciona mediante plantillas de configuración.
- **Pros**: rendimiento bruto imbatible; miles de opciones de configuración mediante
  anotaciones (YAML).
- **Contras**: la configuración puede ser compleja y las anotaciones difíciles de leer.

## Traefik Proxy

Ingress Controller de nueva generación escrito en Go, diseñado específicamente para la era de
los contenedores y la cultura cloud native.

- **Arquitectura**: completamente dinámica. Se conecta directamente a la API de Kubernetes
  (mediante *providers*).
- **Pros**:
  - *Dynamic discovery*: descubre servicios automáticamente.
  - Soporte TLS nativo: genera y renueva certificados SSL sin herramientas externas.
  - Concepto de **middlewares**: permite encadenar peticiones de forma muy limpia.
- **Contras**: su sintaxis nativa difiere de la de Kubernetes.

## cert-manager

En entornos profesionales. Es un **operador de Kubernetes** especializado en la gestión de
certificados de seguridad. Automatiza el ciclo de vida completo:

1. Solicita el certificado SSL (a Let's Encrypt, Vault, etc.).
2. Valida que el dominio sea tuyo.
3. Guarda las claves de forma segura en el clúster (**Secrets**).
4. Las renueva antes de que caduquen.

## Exponer sin abrir puertos: Cloudflare Tunnel

Alternativa al par LoadBalancer + Ingress cuando no puedes (o no quieres) abrir puertos en el
router: un agente dentro del clúster abre una conexión **de salida** y Cloudflare le entrega
el tráfico por ahí.

```
   LoadBalancer + Ingress                 Cloudflare Tunnel
   ──────────────────────                 ─────────────────
   Internet ──► IP pública :443           cloudflared ──(saliente)──► Cloudflare
            (puerto abierto en el                                      ▲
             router / firewall)            Internet ─────► app.tudominio.com
                  │                                                    │
                  ▼                        el tráfico baja por la conexión ya abierta
             Ingress ─► Service            Ingress / Service
```

- **cloudflared** se despliega como un Deployment más y se autentica con un token guardado en
  un Secret.
- El nombre público se gestiona en Cloudflare, no hace falta DNS dinámico ni IP fija.
- Sin puertos de entrada: nada escucha desde fuera, lo que reduce mucho la superficie expuesta
  (ver [puertos que no conviene exponer](../redes/02-puertos-comunes.md)).

| A favor | En contra |
|---|---|
| No hay que abrir puertos ni tener IP fija | dependes de un tercero y de su disponibilidad |
| TLS y protección delante, sin configurarlos | el tráfico se descifra en Cloudflare |
| Ideal para un laboratorio en casa | menos control que un Ingress propio |

Ideas parecidas: **Tailscale Funnel**, **ngrok** o un túnel SSH inverso contra un VPS
(ver [túneles SSH](../seguridad/06-tuneles-ssh-x11-socks.md)). En producción de empresa, lo
habitual sigue siendo Ingress Controller + certificados gestionados con cert-manager.
