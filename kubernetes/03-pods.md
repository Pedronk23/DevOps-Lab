# Kubernetes — Pod

## Qué es

Es la **unidad mínima desplegable** en Kubernetes. Kubernetes no trabaja directamente con
contenedores, sino con pods: un pod es una envoltura que contiene uno o más contenedores
que comparten:

| Recurso compartido | Detalle |
|---|---|
| **Red** | misma IP; los contenedores se comunican entre sí por `localhost` |
| **Almacenamiento** | pueden compartir volúmenes |
| **Ciclo de vida** | se crean, escalan y mueren juntos |

```
                 ┌──────────── POD ────────────┐
                 │  IP: 10.244.1.7             │
                 │                             │
                 │  ┌───────────┐ ┌──────────┐ │
                 │  │ app       │ │ sidecar  │ │
                 │  │ :8080     │ │ :9090    │ │
                 │  └─────┬─────┘ └────┬─────┘ │
                 │        └─ localhost ─┘      │
                 │                             │
                 │  volumen compartido /data   │
                 └─────────────────────────────┘
```

## 1 pod = 1 contenedor (lo más común)

A veces se usa el patrón **sidecar**:

```
Pod
├── Contenedor principal (tu app, p. ej. Node.js)
└── Sidecar (agente que recoge los logs)
```

Otros patrones multi-contenedor:

| Patrón | Para qué |
|---|---|
| **sidecar** | añade una capacidad al principal (logs, métricas, proxy de service mesh) |
| **ambassador** | proxy hacia el exterior; la app habla a `localhost` y el ambassador enruta |
| **adapter** | normaliza la salida del principal (por ejemplo, formato de métricas) |
| **initContainers** | se ejecutan **antes** y hasta completarse: esperar a una BBDD, migrar esquema, descargar config |

## Características

- **Son efímeros**: si mueren, no vuelven solos (para eso están los Deployments).
- Tienen IP propia dentro del clúster, pero **no es fija**.
- Se suelen crear a través de un Deployment que los gestiona.

## Ciclo de vida

```
   Pending  ──►  Running  ──┬──►  Succeeded   (terminó bien; típico de Jobs)
      │                     └──►  Failed      (terminó con error)
      │
      └──► (no hay nodo, imagen no baja, recursos insuficientes…)

                             Unknown  (el nodo no responde)
```

Estados de contenedor que verás en `kubectl get pods`:

| Estado | Significado habitual |
|---|---|
| `ContainerCreating` | bajando imagen o montando volúmenes |
| `ImagePullBackOff` | no puede descargar la imagen (nombre, tag o credenciales del registro) |
| `CrashLoopBackOff` | el contenedor arranca y muere en bucle: mira los logs |
| `Completed` | terminó su función (correcto en Jobs, sospechoso en un Deployment) |
| `OOMKilled` | lo mató el kernel por superar el límite de memoria |
| `Evicted` | el nodo se quedó sin recursos y expulsó el pod |

## Probes (sondas)

```
             ┌──────────────────────────────────────────────┐
  startup ──►│ ¿ha arrancado ya? (desactiva las otras dos)  │
             ├──────────────────────────────────────────────┤
  readiness ►│ ¿puede recibir tráfico? → entra/sale del SVC │
             ├──────────────────────────────────────────────┤
  liveness  ►│ ¿está vivo? → si falla, reinicia el contenedor│
             └──────────────────────────────────────────────┘
```

```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 3
```

> Error clásico: poner una `livenessProbe` demasiado agresiva en una app de arranque lento →
> se reinicia sola en bucle. Para eso está `startupProbe`.

## Recursos: requests y limits

```yaml
resources:
  requests:        # lo que el scheduler reserva para decidir el nodo
    cpu: "100m"
    memory: "128Mi"
  limits:          # el techo duro
    cpu: "500m"
    memory: "256Mi"
```

- Superar el límite de **memoria** → `OOMKilled`.
- Superar el límite de **CPU** → *throttling* (va lento, no muere).

## Ejemplo de manifiesto

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mi-pod
  labels:
    app: mi-app
spec:
  containers:
    - name: mi-app
      image: nginx:1.27
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          memory: "128Mi"
```

## Comandos del día a día

```bash
kubectl get pods -o wide                 # incluye nodo e IP
kubectl describe pod mi-pod              # eventos: por qué no arranca
kubectl logs mi-pod -c mi-app --tail=50
kubectl logs mi-pod --previous           # logs del contenedor que murió
kubectl exec -it mi-pod -- sh
kubectl delete pod mi-pod                # si lo gestiona un Deployment, vuelve a nacer
```
