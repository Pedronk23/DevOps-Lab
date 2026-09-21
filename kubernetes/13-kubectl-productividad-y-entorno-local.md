# Kubernetes — Entorno local y productividad con kubectl

> Objetivo a medio plazo: certificación **CKA** (*Certified Kubernetes Administrator*). El
> examen es práctico y con el tiempo justo: casi todo lo de esta nota está pensado para ir
> rápido.

## Montar un clúster en local

| Opción | Qué es |
|---|---|
| **Rancher Desktop** | clúster local con **k3s** y runtime de contenedores incluido, con interfaz gráfica |
| **k3d** / **kind** | clústeres dentro de contenedores, muy rápidos de crear y destruir |
| **minikube** | el clásico: una VM o contenedor con un clúster de un nodo |
| **Docker Desktop** | trae un Kubernetes de un nodo, suficiente para probar manifiestos |

Otras piezas del entorno:

| Herramienta | Para qué |
|---|---|
| **Homebrew** | gestor de paquetes en macOS y Linux: `brew install kubectl k9s helm` |
| **HAProxy** | balanceador delante de los nodos del control plane en clústeres HA |
| **tmux** | varias shells en una ventana; imprescindible por SSH (ver [atajos de consola](../linux-shell/02-atajos-consola.md)) |
| **htop** | monitor de procesos interactivo: el "K9s" del sistema operativo |
| **vim** | editar manifiestos directamente en el servidor |

## Alias y autocompletado

```bash
# ~/.zshrc o ~/.bashrc
alias k=kubectl
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods -A'

source <(kubectl completion zsh)     # en bash: kubectl completion bash
compdef k=kubectl                    # que el autocompletado funcione también con "k"
```

```bash
alias | grep kubectl        # ver los alias que ya tienes definidos
```

> Guardar el `.zshrc` en un repositorio de git te lleva los alias a cualquier máquina
> (ver [shell y zsh](../linux-shell/01-shell-y-zsh.md)).

**Trampa**: los alias solo existen en tu shell interactiva. Un comando que abre otra shell no
los conoce:

```bash
watch -n 1 "kubectl get pods"    # ✔
watch -n 1 "k get pods"          # ✘ "k: command not found"
```

## Ayuda integrada

```bash
k run --help | less              # -h también vale
k create deployment -h | less
k explain pods                   # documentación de los campos del recurso
k explain pod.spec.containers    # bajando por el árbol
k explain ingress --recursive | less
k api-resources                  # todos los kinds, con su abreviatura y su apiVersion
```

`less` se maneja como vim: `j`/`k` para moverse, `/` para buscar, `q` para salir.

> `k explain` sale del **propio clúster**: siempre corresponde a la versión que tienes
> delante, a diferencia de la documentación web.

## Generar YAML sin crear nada

```bash
k run web --image=nginx --dry-run=client -o yaml > pod.yaml
```

| Parte | Qué hace |
|---|---|
| `--dry-run=client` | simula la creación en local, **no toca el clúster** |
| `--dry-run=server` | la valida el API Server (detecta errores de admisión), pero tampoco la crea |
| `-o yaml` | imprime el manifiesto resultante |

Plantillas que más se usan:

```bash
k run web --image=nginx --dry-run=client -o yaml > pod.yaml
k create deploy web --image=nginx --replicas=3 --dry-run=client -o yaml > deploy.yaml
k create svc clusterip web --tcp=80:8080 --dry-run=client -o yaml > svc.yaml
k create ns miapp --dry-run=client -o yaml > ns.yaml
k create cm config --from-file=app.conf --dry-run=client -o yaml > cm.yaml
k create secret generic db --from-literal=password=s3cr3t --dry-run=client -o yaml > secret.yaml
```

> Es la forma más rápida de partir de un esqueleto válido en vez de escribirlo de memoria.
> Sobre lo que guarda de verdad un Secret: [base64](../fundamentos/base64.md).

## `create` vs `apply`

| Comando | Estilo | Comportamiento |
|---|---|---|
| `k create -f fichero.yaml` | imperativo | falla si el recurso ya existe |
| `k apply -f fichero.yaml` | declarativo | lo crea si no está y aplica **solo los cambios** si ya está |
| `k replace -f fichero.yaml` | imperativo | sustituye el objeto entero |
| `k delete -f fichero.yaml` | — | borra lo que declare el fichero |

```bash
k diff -f deploy.yaml        # qué cambiaría antes de aplicarlo
k apply -f .                 # todos los manifiestos de la carpeta
k apply -k overlays/prod     # kustomize
```

> Regla práctica: **`apply` siempre**. Un objeto creado con `create` no guarda la anotación de
> última configuración aplicada, y el primer `apply` posterior puede comportarse de forma rara.

## Namespaces

Separación lógica dentro del clúster: nombres, cuotas y permisos van por namespace. Buena
práctica: **una aplicación por namespace**.

```bash
k create namespace miapp
k get pods -n miapp
k get pods -A                                    # todos los namespaces
k config set-context --current --namespace=miapp # fijar el namespace por defecto
k config view --minify | grep namespace          # ¿en cuál estoy?
```

| Cuidado | Detalle |
|---|---|
| Recursos sin namespace | nodos, PersistentVolumes, StorageClasses, ClusterRoles, CRDs |
| Borrado | `k delete ns miapp` borra **todo** lo que hay dentro |
| Permisos | los `Role` y `RoleBinding` son por namespace (ver [RBAC](09-rbac.md)) |
| DNS | un Service se resuelve como `servicio.namespace.svc.cluster.local` |

## Vim para YAML

| Atajo | Acción |
|---|---|
| `:set paste` | pegar sin que vim destroce la indentación |
| `i` | modo inserción |
| `Ctrl + U` / `Ctrl + D` | media página arriba / abajo |
| `dd` / `p` | cortar línea / pegarla |
| `:set number` | números de línea, para localizar los errores del parser |
| `:q` / `:wq` / `:q!` | salir / guardar y salir / salir sin guardar |

> La indentación mal puesta es la causa número uno de manifiestos rechazados:
> `k apply --dry-run=server -f fichero.yaml` los caza antes de tocar nada
> (ver [parsers y codificación](../fundamentos/parsers-y-codificacion.md)).
