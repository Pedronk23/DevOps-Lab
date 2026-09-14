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
