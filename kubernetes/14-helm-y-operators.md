# Kubernetes — Helm y Operators

## Helm: el gestor de paquetes de Kubernetes

Un **chart** es un paquete de manifiestos con plantillas y valores configurables. Helm es un
binario que corre en **tu máquina**: se conecta al *current context* de kubectl y aplica los
recursos por ti.

```
   chart (plantillas)  +  values.yaml (tus valores)
              │
              ▼  helm template / install
     manifiestos YAML ya renderizados
              │
              ▼
        API Server  ──►  recursos en el clúster
              │
              ▼
        release (nombre + revisión + historial)
```

| Concepto | Qué es |
|---|---|
| **Chart** | el paquete: plantillas, valores por defecto y metadatos |
| **Repositorio** | servidor donde se publican charts (también valen registros OCI y Nexus) |
| **Release** | una instalación concreta de un chart, con su nombre y su historial |
| **Values** | los parámetros que rellenan las plantillas |
| **Revisión** | cada `install` o `upgrade` crea una nueva, y se puede volver atrás |

## Comandos

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm search repo kube-prometheus-stack --versions

# instalar
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace

# actualizar con tus valores
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring -f values.yaml --version 65.1.0

helm list -A                        # releases de todos los namespaces
helm history monitoring -n monitoring
helm rollback monitoring 1 -n monitoring
helm uninstall monitoring -n monitoring
```

| Opción | Para qué |
|---|---|
| `--install` (junto a `upgrade`) | instala si no existe: el comando queda idempotente |
| `-f values.yaml` | tus valores (se pueden encadenar varios `-f`) |
| `--set clave=valor` | un valor suelto, sin fichero |
| `--version` | **fija la versión del chart**: sin esto, cada despliegue puede traer otra |
| `--atomic` | si falla, deshace el cambio automáticamente |
| `--dry-run --debug` | ver qué haría sin aplicarlo |
| `--create-namespace` | crear el namespace si no existe |

### Partir de los valores por defecto

```bash
helm show values prometheus-community/kube-prometheus-stack > values.yaml
helm template monitoring prometheus-community/kube-prometheus-stack -f values.yaml | less
```

> Flujo recomendado: `show values` → recortar a lo que de verdad cambias → guardar ese
> `values.yaml` **en git** → desplegar siempre con `helm upgrade --install -f values.yaml`.
> El `values.yaml` completo, de miles de líneas, no se versiona: se vuelve ilegible.

`helm upgrade` con valores nuevos actualiza los recursos y **recrea los pods afectados**: es
un despliegue, no un cambio silencioso.

## Operators y CRDs

Un **CRD** (*CustomResourceDefinition*) añade un tipo de recurso nuevo a la API de Kubernetes.
Un **Operator** es un controlador que vigila esos recursos y actúa: instala, configura, repara
y actualiza una aplicación como lo haría un administrador.

```
   Tú creas:  ServiceMonitor (CRD)
                    │
                    ▼
   Prometheus Operator lo ve  ──►  reconfigura Prometheus  ──►  raspa el servicio
```

```bash
k get crd                                  # qué CRDs hay instalados
k api-resources --api-group=monitoring.coreos.com
k explain servicemonitor.spec
```

| Operator | Recursos que aporta |
|---|---|
| **Prometheus Operator** | `ServiceMonitor`, `PodMonitor`, `PrometheusRule` (ver [observabilidad](../observabilidad/prometheus-y-grafana.md)) |
| **cert-manager** | `Certificate`, `Issuer`, `ClusterIssuer` (ver [Ingress Controllers y cert-manager](07-ingress-controllers-y-cert-manager.md)) |
| **External Secrets** | `ExternalSecret`, `SecretStore` (ver [Vault](../vault/01-vault-kv.md)) |
| Operators de bases de datos | clústeres de PostgreSQL o MySQL con backups y failover |

Los kinds más habituales, en la [chuleta de kinds abreviados](11-kinds-abreviados.md).

## Errores típicos

| Error | Causa / solución |
|---|---|
| `cannot re-use a name that is still in use` | ya existe una release con ese nombre: `helm upgrade --install`, o desinstalarla |
| `another operation (install/upgrade) is in progress` | una release se quedó a medias: `helm rollback` a la última revisión buena |
| Tras `helm uninstall` quedan CRDs | Helm **no borra CRDs** por diseño: se borran a mano, y con ellos sus recursos |
| Cambia el comportamiento sin tocar nada | no fijaste `--version` y el repositorio trajo un chart nuevo |
| Un valor del `values.yaml` no surte efecto | la clave está mal anidada: comprobar con `helm template` o `helm get values <release>` |
