# Kubernetes

Notas sobre orquestación de contenedores: arquitectura, workloads, red, almacenamiento,
seguridad y herramientas de operación.

## Orden de lectura recomendado

| # | Nota | Qué cubre |
|---|---|---|
| 01 | [Arquitectura](01-arquitectura.md) | control plane, worker nodes, bucle de reconciliación |
| 02 | [kubeconfig y kubectl](02-kubeconfig-y-kubectl.md) | cómo hablas con el clúster |
| 03 | [Pods](03-pods.md) | unidad mínima, sidecars, ciclo de vida, probes |
| 04 | [Deployment y ReplicaSet](04-deployments-y-replicasets.md) | réplicas, rollouts, rollbacks |
| 05 | [Service e Ingress](05-services-e-ingress.md) | descubrimiento y exposición |
| 06 | [Networking (1): LoadBalancer y MetalLB](06-networking-loadbalancer-metallb.md) | exposición on-premise |
| 07 | [Networking (2): Ingress Controllers y cert-manager](07-ingress-controllers-y-cert-manager.md) | nginx vs Traefik, TLS automático |
| 08 | [Almacenamiento: PV, PVC y Longhorn](08-almacenamiento-longhorn.md) | persistencia |
| 09 | [RBAC](09-rbac.md) | permisos e identidades |
| 10 | [K9s](10-k9s.md) | TUI de operación diaria |
| 11 | [Kinds abreviados](11-kinds-abreviados.md) | chuleta de recursos |
| 12 | [Networking (3): resumen con CNI](12-networking-resumen-cni.md) | visión global del tráfico |
