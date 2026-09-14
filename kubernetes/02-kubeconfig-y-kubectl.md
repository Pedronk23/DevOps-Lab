# Kubernetes — kubeconfig y kubectl

## Qué es un kubeconfig

Archivo de texto (formato YAML) que contiene las credenciales y datos de conexión para
hablar con un clúster de Kubernetes.

**Contiene:**

| Sección | Qué guarda |
|---|---|
| `clusters` | la dirección del servidor (API Server) y su certificado de CA |
| `users` | tus credenciales (certificado de cliente, token o similar) |
| `contexts` | la combinación de "qué clúster + qué usuario + qué namespace" |

> ⚠️ **Es información sensible.** Equivale a una llave de acceso al clúster.
> No lo compartas: trátalo como una contraseña.

`kubectl` lo busca por defecto en:

```
~/.kube/config
```

## kubectl

Es la herramienta en línea de comandos de Kubernetes. Permite interactuar con el clúster
para gestionar aplicaciones y recursos: habla con la API de Kubernetes y te deja:

| Acción | Ejemplo |
|---|---|
| Desplegar aplicaciones | `kubectl apply -f app.yml` |
| Ver el estado de pods, servicios, deployments | `kubectl get pods` |
| Depurar problemas | `kubectl logs`, `kubectl exec` |
| Escalar réplicas | `kubectl scale deployment mi-app --replicas=3` |
| Gestionar configuración y secretos | `kubectl create secret …` |

### Sintaxis general

```
kubectl [comando] [tipo-de-recurso] [nombre] [flags]
```

> Kubernetes es el sistema que orquesta tus contenedores; kubectl es el mando a distancia
> con el que lo controlas.
