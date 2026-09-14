# Kubernetes — Deployment y ReplicaSet

## Relación

```
Deployment          "quiero 3 réplicas de nginx:1.25, actualizadas sin downtime"
└── ReplicaSet (v1) "garantizo que existan 3 pods con este template"
    ├── Pod
    ├── Pod
    └── Pod
```

Cuando actualizas la imagen, Kubernetes crea un **nuevo ReplicaSet**:

```
Deployment
├── ReplicaSet (v1) → 0 pods  (el viejo, apagándose)
└── ReplicaSet (v2) → 3 pods  (el nuevo, arrancando)
```

Esto es lo que permite los **rollouts sin downtime** y los **rollbacks**.

## ReplicaSet

Su único trabajo es **garantizar que siempre haya N pods corriendo**. Nada más.

- Si un Pod muere → lo recrea.
- Si hay de más → mata el sobrante.
- No sabe nada de actualizaciones ni de historial.

> En la práctica nunca creas ReplicaSets a mano: los gestiona el Deployment.

## Deployment

Es la capa que gestiona por encima del ReplicaSet. Añade:

| Capacidad | Qué aporta |
|---|---|
| **Rollouts** | actualiza pods gradualmente, sin downtime |
| **Rollbacks** | vuelve a una versión anterior con un comando |
| **Historial** | guarda las revisiones anteriores |
| **Estrategias** | controla cómo se hace la actualización |

## Estrategias de actualización

```
RollingUpdate (por defecto)        Recreate
──────────────────────────         ───────────────────
[v1][v1][v1]                       [v1][v1][v1]
[v1][v1][v2]  ← sube uno nuevo          ↓  mata todo
[v1][v2][v2]     y baja uno viejo  [  ][  ][  ]  ← DOWNTIME
[v2][v2][v2]                            ↓
sin downtime                       [v2][v2][v2]
```

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1   # cuántos pods pueden faltar durante el rollout
    maxSurge: 1         # cuántos pods extra se permiten temporalmente
```

| Estrategia | Cuándo usarla |
|---|---|
| `RollingUpdate` | por defecto: apps web, APIs, cualquier cosa stateless |
| `Recreate` | cuando dos versiones no pueden coexistir (migración de esquema incompatible, licencia de un solo proceso) |

## Flujo de actualización

Pasas de `nginx:1.25` a `nginx:1.26` y haces `kubectl apply`:

```
[v1][v1][v1] → [v1][v1][v2] → [v1][v2][v2] → [v2][v2][v2]  (actualización completa)
```

Si algo falla, Kubernetes **para el rollout** y puedes revertir.

## Manifiesto de ejemplo

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-app
spec:
  replicas: 3
  revisionHistoryLimit: 5        # cuántos ReplicaSets viejos guarda
  selector:
    matchLabels:
      app: mi-app                # DEBE coincidir con las labels del template
  template:
    metadata:
      labels:
        app: mi-app
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
```

> El `selector` es inmutable: si necesitas cambiarlo, hay que borrar y recrear el Deployment.

## Comandos del día a día

```bash
kubectl rollout status deployment/mi-app        # seguir el despliegue
kubectl rollout history deployment/mi-app       # ver revisiones
kubectl rollout undo deployment/mi-app          # volver a la anterior
kubectl rollout undo deployment/mi-app --to-revision=3
kubectl rollout restart deployment/mi-app       # reiniciar pods sin cambiar la imagen
kubectl scale deployment/mi-app --replicas=5
kubectl set image deployment/mi-app web=nginx:1.27
```

## Cuándo NO usar un Deployment

| Necesidad | Recurso correcto |
|---|---|
| Identidad y disco estables por réplica (BBDD, Kafka, etcd) | **StatefulSet** |
| Un pod en cada nodo (agentes de logs, red, monitorización) | **DaemonSet** |
| Tarea que corre una vez y termina | **Job** |
| Tarea periódica | **CronJob** |

## Escalado automático (HPA)

```
  métricas (CPU/memoria/custom)
            │
            ▼
   ┌──────────────────┐        ajusta
   │ HorizontalPod    │──────► replicas del Deployment
   │ Autoscaler       │
   └──────────────────┘
```

```bash
kubectl autoscale deployment mi-app --min=2 --max=10 --cpu-percent=70
```

> El HPA necesita `requests` definidos y el *metrics-server* instalado en el clúster.
