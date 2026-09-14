# Kubernetes Networking (1) — Capa de red externa: LoadBalancer y MetalLB

Cuando montas un clúster de Kubernetes, los pods viven en una red interna privada. Para que
alguien desde fuera pueda acceder a ellos necesitas abrir una puerta al exterior. Ahí trabaja
esta capa.

## LoadBalancer (el concepto de servicio)

Un LoadBalancer es un objeto de tipo Service. Es una abstracción que le dice al clúster:
*"quiero exponer esta app al mundo a través de una dirección IP pública y única"*.

### En la nube

Si estás en AWS (EKS) o Google Cloud (GKE), Kubernetes se comunica mediante APIs con el
proveedor. La nube crea automáticamente un balanceador de carga físico/virtual fuera del
clúster (un ALB de AWS, por ejemplo), le asigna una IP pública de internet y redirige el
tráfico hacia tus nodos.

### On-premise

Si montas el clúster en tus propios servidores locales, Kubernetes no tiene una API de nube
a la que pedirle una IP. El servicio se queda eternamente en `<pending>`.
Para solucionarlo necesitas **MetalLB**.

## MetalLB (el proveedor de IPs local)

Controlador de red diseñado específicamente para entornos donde **no hay un proveedor de
nube**. Su trabajo es simular el comportamiento de un balanceador de carga de la nube en tu
propia red local.

Cuando lo instalas, le entregas un **pool** (rango) de IPs libres de tu red local
(ej.: `192.168.1.200` – `192.168.1.250`). Cuando un servicio pide un LoadBalancer, MetalLB
toma una de esas IPs y se la asigna.

### Modos de funcionamiento

Para anunciar a la red física que esa IP ahora pertenece al clúster, MetalLB usa dos modos:

**Modo Layer 2 (L2)**
El más sencillo. Uno de los nodos del clúster "se adueña" de la IP mediante el protocolo
ARP (IPv4) o NDP (IPv6). El inconveniente: todo el tráfico entra por ese único nodo.
Si falla, MetalLB asigna la IP a otro nodo.

**Modo BGP (Border Gateway Protocol)**
Modo profesional para producción. Establece una sesión de enrutamiento dinámico BGP con los
routers físicos de la red. El router se encarga de balancear el tráfico de verdad entre
todas las máquinas del clúster, ofreciendo alta disponibilidad y redundancia real.

## SNAT

Cuando el tráfico entra por MetalLB o un LoadBalancer, Kubernetes hace un proceso llamado
**SNAT** (*Source Network Address Translation*): modifica el paquete de red para que parezca
que proviene del propio nodo de Kubernetes, ocultando la IP real del usuario de internet.

> Efecto secundario práctico: en los logs de la app ves la IP del nodo, no la del cliente.
