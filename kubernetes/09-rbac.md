# Kubernetes — RBAC

**RBAC** = *Role-Based Access Control* (control de acceso basado en roles).
Es el sistema con el que Kubernetes decide **quién puede hacer qué sobre qué recurso de la
API**.

## Las cuatro piezas

```
      ┌──────────────────┐                    ┌──────────────────┐
      │  ServiceAccount  │                    │      Role        │
      │  (quién eres)    │                    │  (qué se puede   │
      │  o usuario/grupo │                    │   hacer)         │
      └────────┬─────────┘                    └─────────┬────────┘
               │                                        │
               └──────────►┌──────────────┐◄────────────┘
                           │ RoleBinding  │
                           │ (el pegamento)│
                           └──────────────┘
```

| Objeto | Qué es | Alcance |
|---|---|---|
| **ServiceAccount (SA)** | la "identidad" que usa un pod o proceso para hablar con la API de Kubernetes | namespace |
| **Role** | una lista de permisos (verbos sobre recursos): "puede `get`/`list` pods", etc. | namespace |
| **RoleBinding** | el "pegamento": asocia un Role a una SA y le concede esos permisos | namespace |
| **ClusterRole / ClusterRoleBinding** | lo mismo pero a nivel de todo el clúster (sin namespace) | clúster |

## Combinaciones posibles

```
Role        + RoleBinding         → permisos en UN namespace
ClusterRole + RoleBinding         → permisos del ClusterRole, limitados a UN namespace
                                    (patrón muy útil: defines el rol una vez y lo reutilizas)
ClusterRole + ClusterRoleBinding  → permisos en TODO el clúster
Role        + ClusterRoleBinding  → NO existe / no válido
```

## Verbos habituales

| Verbo | Acción |
|---|---|
| `get` | leer un recurso concreto |
| `list` | listar recursos |
| `watch` | suscribirse a cambios |
| `create` / `update` / `patch` | escribir |
| `delete` / `deletecollection` | borrar |
| `*` | todos (evítalo salvo en roles de administración) |

Recursos que se suelen olvidar: los **subrecursos** se nombran con barra, como
`pods/log`, `pods/exec` o `deployments/scale`.

## Ejemplo mínimo

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mi-sa
  namespace: mi-namespace
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: mi-namespace
  name: lector-pods
rules:
  - apiGroups: [""]                      # "" = core API group
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: lector-pods-binding
  namespace: mi-namespace
subjects:
  - kind: ServiceAccount
    name: mi-sa
roleRef:
  kind: Role
  name: lector-pods
  apiGroup: rbac.authorization.k8s.io
```

Y el pod que usa esa identidad:

```yaml
spec:
  serviceAccountName: mi-sa
```

## Cómo llega la identidad al pod

```
   ┌──── POD ────────────────────────────────┐
   │ /var/run/secrets/kubernetes.io/         │
   │   serviceaccount/token   ← JWT montado  │──► API Server
   │                                         │      │
   └─────────────────────────────────────────┘      ▼
                                         autenticación → autorización (RBAC)
```

## Verificar permisos

```bash
# ¿puedo yo?
kubectl auth can-i create deployments -n mi-namespace

# ¿puede esa ServiceAccount?
kubectl auth can-i get pods \
  --as=system:serviceaccount:mi-namespace:mi-sa -n mi-namespace

# listar todo lo que puede hacer
kubectl auth can-i --list --as=system:serviceaccount:mi-namespace:mi-sa
```

## Buenas prácticas

- **Mínimo privilegio**: empezar con `Role` en un namespace y subir a `ClusterRole` solo
  cuando el recurso es realmente global (nodos, CRDs, namespaces…).
- Una **SA por aplicación**: no reutilizar la `default` del namespace.
- Desactivar el montaje del token si la app no habla con la API:
  `automountServiceAccountToken: false`.
- Cuidado con `ClusterRoleBinding` a `cluster-admin`: equivale a root en el clúster.
- Revisar quién tiene permisos sobre `secrets`: leer secrets es leer contraseñas.
