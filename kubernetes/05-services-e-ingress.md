# Kubernetes — Service e Ingress

## Service

**El problema que resuelve**: los pods tienen IPs que cambian constantemente, así que
necesitas algo estable para llegar a ellos.

Es una **capa fija estable**: una IP fija y un DNS que siempre apunta a los pods correctos.

```
        Service "mi-app"  (ClusterIP 10.96.0.42, DNS mi-app.default.svc.cluster.local)
                    │  selector: app=mi-app
        ┌───────────┼───────────┐
        ▼           ▼           ▼
   [pod 10.244.1.7][pod .2.3][pod .3.9]     ← las IPs cambian, el Service no
```

- Usa **labels** para saber a qué pods mandar el tráfico.
- Si un pod muere y se recrea con otra IP, el Service lo detecta automáticamente.
- Por debajo mantiene una lista de **Endpoints** (o `EndpointSlice`) con las IPs vivas;
  solo entran los pods que pasan la `readinessProbe`.

## Tipos habituales

```
                     Internet
                        │
                        ▼
        ┌───────────────────────────────┐
        │ LoadBalancer   (IP externa)   │  producción
        ├───────────────────────────────┤
        │ NodePort   nodo:30000-32767   │  desarrollo / acceso tosco
        ├───────────────────────────────┤
        │ ClusterIP   solo interno      │  comunicación entre servicios
        └───────────────────────────────┘
                        │
                       pods
```

| Tipo | Comportamiento |
|---|---|
| **ClusterIP** | solo accesible dentro del clúster. Para comunicación interna entre servicios |
| **NodePort** | abre un puerto (30000-32767) en cada nodo del clúster. Accesible desde fuera, pero de forma tosca. Útil para desarrollo |
| **LoadBalancer** | crea un balanceador de carga externo (en AWS, Azure, o MetalLB on-premise). La forma correcta de exponer en producción |
| **ExternalName** | no balancea nada: devuelve un CNAME a un host externo. Útil para migrar servicios fuera del clúster sin cambiar la app |
| **Headless** (`clusterIP: None`) | sin IP virtual: el DNS devuelve las IPs de los pods. Lo usan los StatefulSets |

## Manifiesto de ejemplo

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mi-app
spec:
  type: ClusterIP
  selector:
    app: mi-app          # debe coincidir con las labels de los pods
  ports:
    - port: 80           # puerto del Service
      targetPort: 8080   # puerto del contenedor
```

> Confusión típica: `port` es por donde escucha el Service; `targetPort`, por donde escucha
> el contenedor; `nodePort`, el puerto que se abre en cada nodo (solo NodePort/LoadBalancer).

## DNS automática

Kubernetes crea automáticamente un DNS para cada Service:

```
mi-servicio.mi-namespace.svc.cluster.local
│           │            │
│           │            └── dominio interno del clúster
│           └── namespace
└── nombre del service
```

En la práctica, desde el mismo namespace basta con:

```
mi-servicio:80
```

Y desde otro namespace: `mi-servicio.otro-namespace`.

## Ingress

Es el **manifiesto YAML donde defines las reglas de enrutamiento** HTTP/HTTPS.
Solo es configuración.

Motivo de existir: para manejar varios servicios, un LoadBalancer por cada uno sale caro y
es difícil de mantener.

```
                        una sola IP pública
                               │
                    ┌──────────▼──────────┐
                    │  Ingress Controller │
                    └──────────┬──────────┘
        app.ejemplo.com  ──────┤
        api.ejemplo.com  ──────┤  lee las reglas del Ingress
        /blog            ──────┘
              │            │            │
              ▼            ▼            ▼
          svc-web      svc-api      svc-blog
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mi-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts: [app.ejemplo.com]
      secretName: app-tls        # lo rellena cert-manager
  rules:
    - host: app.ejemplo.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: mi-app
                port:
                  number: 80
```

## Ingress Controller

Es el **proceso que lee esas reglas y las ejecuta**. Hay que instalarlo aparte.

Los más usados: `ingress-nginx` y **Traefik** (comparativa en
[Networking 2](07-ingress-controllers-y-cert-manager.md)).

## Comandos útiles

```bash
kubectl get svc,endpoints                     # ¿el service tiene endpoints?
kubectl get ingress
kubectl describe ingress mi-ingress
kubectl port-forward svc/mi-app 8080:80       # depurar sin exponer nada
```

> Si un Service "no funciona", el 90 % de las veces es que **no tiene endpoints**: las labels
> del selector no coinciden con los pods, o los pods no pasan la readiness.
