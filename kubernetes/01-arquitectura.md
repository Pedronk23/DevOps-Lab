# Kubernetes — Arquitectura

## Qué es

Kubernetes (K8s) es un **orquestador de contenedores**: un sistema que automatiza el
despliegue, escalado, red y ciclo de vida de aplicaciones empaquetadas en contenedores.
Nació en Google (basado en su sistema interno *Borg*) y se convirtió en open source en 2014.

Si tienes 10, 100 o 1000 máquinas con contenedores, es quien decide dónde van, quién los
reinicia si mueren y quién balancea el tráfico.

## Esquema general

```
┌──────────────────────────── CONTROL PLANE ────────────────────────────┐
│  El cerebro del clúster: solo toma decisiones, NO corre tu aplicación │
│                                                                       │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌─────────────────┐  │
│  │ API SERVER │→ │    ETCD    │→ │ SCHEDULER  │→ │ CONTROLLER MGR  │  │
│  │ puerta de  │  │ BBDD dist. │  │ decide en  │  │ bucles de ctrl. │  │
│  │ entrada    │  │ clave-valor│  │ qué nodo   │  │ ReplicaSet,     │  │
│  │ REST+Watch │  │ estado     │  │ va cada    │  │ Node, Job,      │  │
│  │ kubectl    │  │ deseado    │  │ Pod nuevo  │  │ Endpoint…       │  │
│  │ habla aquí │  │            │  │ CPU/mem/af.│  │                 │  │
│  └────────────┘  └────────────┘  └────────────┘  └─────────────────┘  │
└───────────┬───────────────────┬───────────────────┬───────────────────┘
            ▼                   ▼                   ▼
   ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
   │  WORKER NODE 1   │ │  WORKER NODE 2   │ │  WORKER NODE 3   │
   │ ┌──────────────┐ │ │ ┌──────────────┐ │ │ ┌──────────────┐ │
   │ │   KUBELET    │ │ │ │   KUBELET    │ │ │ │   KUBELET    │ │
   │ ├──────────────┤ │ │ ├──────────────┤ │ │ ├──────────────┤ │
   │ │  KUBE-PROXY  │ │ │ │  KUBE-PROXY  │ │ │ │  KUBE-PROXY  │ │
   │ └──────────────┘ │ │ └──────────────┘ │ │ └──────────────┘ │
   │ [POD A] [POD B]  │ │ [POD C] [POD D]  │ │ [POD E] [POD F]  │
   └──────────────────┘ └──────────────────┘ └──────────────────┘
```

## Control Plane

Es el cerebro. **Nunca corre tu aplicación, solo toma decisiones.**

| Componente | Función |
|---|---|
| **API Server** | Todo pasa por aquí: kubectl, otros componentes y tus propias apps hablan REST con él. Es el único que escribe en etcd |
| **etcd** | Base de datos distribuida clave-valor. Guarda el **estado deseado** de todo el clúster. Si muere, el clúster pierde la memoria |
| **Scheduler** | Cuando aparece un Pod sin nodo asignado, mira los recursos disponibles y las restricciones (afinidad, taints/tolerations) y elige dónde colocarlo |
| **Controller Manager** | Ejecuta bucles de control. Cada controlador observa el estado actual, lo compara con el deseado y actúa para reducir la diferencia |

## Worker Nodes

Donde corre tu código. Cada uno tiene:

| Componente | Función |
|---|---|
| **kubelet** | Agente que habla con el API Server y hace que los Pods existan físicamente en el nodo, usando el runtime de contenedores (containerd, CRI-O) |
| **kube-proxy** | Mantiene las reglas de red (iptables / IPVS) para que los Services funcionen |

## El bucle de reconciliación: el corazón de K8s

```
        ┌──────────────────────────────┐
        │  OBSERVAR                    │◄──────────────┐
        │  ¿cuántos Pods hay ahora?    │               │
        └──────────────┬───────────────┘               │
                       ▼                               │
 ┌───────────────┐  ┌──────────────────┐   ┌───────────────────────┐
 │ ETCD          │  │  DIFERENCIA      │   │  CLÚSTER REAL         │
 │ estado deseado│─▶│  deseado ≠ actual│   │  estado actual        │
 │ replicas: 3   │  └────────┬─────────┘   │  corriendo: 2 pods    │
 └───────────────┘           ▼             └───────────▲───────────┘
                    ┌──────────────────┐               │
                    │  ACTUAR          │───────────────┘
                    │  crear/borrar Pod│
                    └──────────────────┘
```

La idea: Kubernetes no ejecuta órdenes puntuales, **converge** continuamente hacia el estado
declarado en etcd.
