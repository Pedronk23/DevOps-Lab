# Kubernetes — Kinds abreviados (chuleta)

## Workloads (cargas que ejecutan pods)

| Abreviatura | Kind | Qué es |
|---|---|---|
| `pod` | Pod | unidad mínima ejecutable (1+ contenedores) |
| `deploy` | Deployment | gestiona pods sin estado; controla réplicas y rollouts |
| `rs` | ReplicaSet | lo crea un Deployment por debajo: mantiene N réplicas |
| `sts` | StatefulSet | como un Deployment pero con identidad y almacenamiento estables |
| `ds` | DaemonSet | un pod por nodo (ej.: agentes de logging o de red) |
| `job` | Job | tarea puntual que corre hasta completarse (ej.: un init de un servicio) |
| — | ControllerRevision | historial de revisiones de un StatefulSet/DaemonSet (rollbacks) |

## Red y exposición

| Abreviatura | Kind | Qué es |
|---|---|---|
| `svc` | Service | IP/DNS estable que balancea hacia un grupo de pods |
| `ing` | Ingress | reglas HTTP/HTTPS de entrada (host/path → Service) |
| `netpol` | NetworkPolicy | firewall a nivel de pod: qué tráfico se permite |

## Almacenamiento

| Abreviatura | Kind | Qué es |
|---|---|---|
| `pvc` | PersistentVolumeClaim | petición de disco persistente |

## Config y secretos

| Abreviatura | Kind | Qué es |
|---|---|---|
| `cm` | ConfigMap | configuración **no sensible** en clave/valor |
| — | Secret | datos sensibles (password, token, certs) en base64 |

## RBAC / identidad

| Abreviatura | Kind | Qué es |
|---|---|---|
| `sa` | ServiceAccount | identidad de un pod frente a la API |
| `role` | Role | permisos dentro de un namespace |
| `c-role` | ClusterRole | permisos a nivel de todo el clúster |
| `crb` | ClusterRoleBinding | asocia un ClusterRole a una SA o usuario |

## Disponibilidad y prioridad

| Abreviatura | Kind | Qué es |
|---|---|---|
| `pdb` | PodDisruptionBudget | mínimo de pods que deben seguir vivos durante mantenimiento/drenado |
| `pc` | PriorityClass | prioridad de scheduling (a quién se desaloja antes si faltan recursos) |

## Extensiones / CRDs

Recursos que añaden los operadores instalados en el clúster.

| Kind | Qué es |
|---|---|
| `crd` (CustomResourceDefinition) | mecanismo que define nuevos kinds en el clúster |
| `Certificate` | de cert-manager: pide y renueva un certificado TLS |
| `ExternalSecret` | de External Secrets Operator (ESO): sincroniza un Secret desde un backend externo |
| `Password` | generador de ESO: produce una contraseña aleatoria para materializarla en un Secret |
