# Prometheus, Node Exporter y Grafana

## Observabilidad: los tres pilares

| Pilar | Responde a | Herramientas habituales |
|---|---|---|
| **Métricas** | ¿cuánto? ¿con qué tendencia? (CPU al 90%, 200 peticiones/s) | Prometheus, VictoriaMetrics, Mimir |
| **Logs** | ¿qué pasó exactamente? (el mensaje de error) | Loki, Elasticsearch/OpenSearch, Graylog |
| **Trazas** | ¿por dónde pasó esta petición y dónde tardó? | Tempo, Jaeger, Zipkin |

- **OpenTelemetry** es el estándar abierto para **generar y transportar** los tres tipos de
  datos. No almacena nada: los envía a los backends de la tabla.
- **Grafana** es la capa de visualización común a todos ellos.

> Monitorización te dice **que** algo va mal; observabilidad te permite averiguar **por qué**
> sin haberlo previsto de antemano.

## Prometheus

Sistema de monitorización **por pull**: cada cierto tiempo va "raspando" (*scraping*) endpoints
HTTP que devuelven métricas. Solo sabe hacer peticiones HTTP y guardar los números que recibe
en su base de datos de series temporales.

### Pull vs push

```
   PULL (Prometheus)                         PUSH (InfluxDB, Graphite…)
   ─────────────────                         ──────────────────────────
   Prometheus ──GET /metrics──► exporter     agente ──envía datos──► servidor
   · el servidor decide la frecuencia        · el agente decide cuándo
   · si el target no responde → up == 0      · si el agente muere, simplemente
     (sabes que está caído)                    dejan de llegar datos
   · el target solo expone un endpoint       · atraviesa NAT/firewalls más fácil
```

Para trabajos cortos (un cron, un job de CI) que terminan antes del siguiente scrape existe
**Pushgateway**: el job empuja sus métricas allí y Prometheus las raspa después.

### Arquitectura

```
   ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
   │ node_exporter│  │ app con      │  │ blackbox_exporter│   ← targets (exponen /metrics)
   │ :9100        │  │ /metrics     │  │ :9115            │
   └──────▲───────┘  └──────▲───────┘  └────────▲─────────┘
          │  scrape         │                   │
          └─────────────────┼───────────────────┘
                            │      service discovery
                   ┌────────┴─────────┐   (ficheros, Kubernetes,
                   │   PROMETHEUS     │    DNS, Consul, cloud…)
                   │   :9090          │
                   │  TSDB + PromQL   │
                   │  reglas alertas  │
                   └───┬──────────┬───┘
            consultas  │          │ alertas disparadas
                       ▼          ▼
               ┌──────────┐  ┌──────────────┐
               │ GRAFANA  │  │ ALERTMANAGER │──► correo, Slack, Telegram, PagerDuty
               │ :3000    │  │ :9093        │   (agrupa, silencia, enruta)
               └──────────┘  └──────────────┘
```

## Node Exporter

Es justamente uno de esos endpoints. Es un **binario oficial** (escrito en Go, sin
dependencias) que, una vez arrancado, levanta un servidor HTTP (por defecto en el puerto
**9100**) y publica en la ruta `/metrics` un montón de datos del sistema: uso de CPU, memoria,
disco, red, temperatura…

### Instalarlo como servicio (Linux)

```bash
VERSION=1.8.2      # revisar la última en github.com/prometheus/node_exporter/releases
curl -fLO https://github.com/prometheus/node_exporter/releases/download/v${VERSION}/node_exporter-${VERSION}.linux-amd64.tar.gz
tar xzf node_exporter-${VERSION}.linux-amd64.tar.gz
sudo mv node_exporter-${VERSION}.linux-amd64/node_exporter /usr/local/bin/
sudo useradd --system --no-create-home --shell /usr/sbin/nologin node_exporter
```

```ini
# /etc/systemd/system/node_exporter.service
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
ExecStart=/usr/local/bin/node_exporter --collector.systemd
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
curl -s localhost:9100/metrics | grep -E '^node_(load1|memory_MemAvailable_bytes)'
```

> En Windows el equivalente es **windows_exporter** (puerto **9182**), que se instala como MSI
> y expone CPU, disco, servicios, IIS, etc.

### Qué devuelve `/metrics`

```
# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 184523.62
node_cpu_seconds_total{cpu="0",mode="user"} 3120.45
│                     │                     │
│                     │                     └── valor
│                     └── etiquetas (labels): cada combinación es una SERIE distinta
└── nombre de la métrica
```

## Flujo completo

```
Node Exporter lee /proc, /sys y otras fuentes del kernel
        │  y traduce esa información a métricas
        ▼
Prometheus consulta cada X segundos el /metrics de ese exporter
        │
        ▼
Los datos se almacenan en la BBDD de series temporales
        │
        ▼
Normalmente los visualizas en Grafana con dashboards
```

## Tipos de métricas

| Tipo | Qué es | Ejemplo | Cómo se consulta |
|---|---|---|---|
| **Counter** | solo sube (se reinicia a 0 si el proceso reinicia) | `http_requests_total` | siempre con `rate()` o `increase()` |
| **Gauge** | sube y baja | `node_memory_MemAvailable_bytes` | directamente |
| **Histogram** | distribución en "cubos" (`_bucket`, `_sum`, `_count`) | `http_request_duration_seconds` | `histogram_quantile()` |
| **Summary** | percentiles calculados en el cliente | `go_gc_duration_seconds` | directamente (no se agrega bien entre instancias) |

> **Cardinalidad**: cada combinación de etiquetas es una serie nueva. Una etiqueta con el ID
> de usuario, la IP del cliente o la URL completa genera millones de series y tumba
> Prometheus. Etiquetas sí para valores **acotados** (entorno, método, código HTTP).

## Configuración: `prometheus.yml`

```yaml
global:
  scrape_interval: 15s          # cada cuánto raspa (por defecto 1m)
  evaluation_interval: 15s      # cada cuánto evalúa las reglas

rule_files:
  - /etc/prometheus/rules/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["localhost:9093"]

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node
    static_configs:
      - targets: ["srv01.lab.local:9100", "srv02.lab.local:9100"]
        labels:
          entorno: lab

  - job_name: node-ficheros            # descubrimiento por ficheros: añadir hosts sin reiniciar
    file_sd_configs:
      - files: ["/etc/prometheus/targets/*.yml"]
```

```bash
promtool check config /etc/prometheus/prometheus.yml     # validar antes de recargar
promtool check rules /etc/prometheus/rules/*.yml
curl -X POST http://localhost:9090/-/reload              # recarga en caliente (requiere --web.enable-lifecycle)
```

- `http://prometheus:9090/targets` → estado de cada target (**UP**/**DOWN** y el error).
- Retención por defecto: **15 días** (`--storage.tsdb.retention.time=30d` para cambiarla).
  Para meses o años: Thanos, Mimir o VictoriaMetrics.

## PromQL: lo imprescindible

### Selectores

```promql
up                                          # todas las series de "up"
up{job="node"}                              # etiqueta exacta
up{instance=~"web.*"}                       # regex (RE2, anclada: debe casar entera)
node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}
http_requests_total[5m]                     # vector de rango: los valores de los últimos 5 min
```

### Funciones y agregaciones

| Función | Para qué |
|---|---|
| `rate(counter[5m])` | incremento **por segundo**, suavizado en la ventana |
| `irate(counter[5m])` | por segundo usando solo las 2 últimas muestras (picos, gráficas finas) |
| `increase(counter[1h])` | cuánto ha subido en la ventana |
| `sum by (label)` / `avg` / `max` / `min` / `count` | agregar series |
| `topk(5, …)` | las 5 mayores |
| `predict_linear(gauge[6h], 86400)` | extrapolar el valor dentro de 24 h |
| `histogram_quantile(0.95, …)` | percentil 95 a partir de un histograma |
| `absent(metrica)` | devuelve 1 si la métrica **no existe** (target desaparecido) |

> Regla práctica: la ventana de `rate()` debe ser al menos **4 veces** el `scrape_interval`
> (con 15 s, `[1m]` como mínimo; `[5m]` es lo habitual).

### Consultas típicas de nodo

```promql
# ¿qué targets están caídos?
up == 0

# % de CPU usada por instancia
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# % de memoria usada
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# % de disco usado
100 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}
       / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"} * 100)

# ¿se llenará el disco en las próximas 24 h?
predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}[6h], 24 * 3600) < 0

# carga relativa al número de CPUs
node_load5 / count by (instance) (node_cpu_seconds_total{mode="idle"})

# tráfico de red recibido en Mbit/s
rate(node_network_receive_bytes_total{device!~"lo|veth.*"}[5m]) * 8 / 1e6
```

### Consultas típicas de aplicación (método RED)

```promql
# Rate: peticiones por segundo
sum by (job) (rate(http_requests_total[5m]))

# Errors: % de respuestas 5xx
sum(rate(http_requests_total{code=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100

# Duration: latencia p95
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```

| Método | Para | Qué medir |
|---|---|---|
| **USE** | recursos (CPU, disco, red) | *Utilization*, *Saturation*, *Errors* |
| **RED** | servicios (APIs, webs) | *Rate*, *Errors*, *Duration* |
| **4 Golden Signals** (Google SRE) | cualquier servicio | latencia, tráfico, errores, saturación |

## Alertas

### Reglas en Prometheus

```yaml
# /etc/prometheus/rules/nodos.yml
groups:
  - name: nodos
    rules:
      - alert: InstanciaCaida
        expr: up == 0
        for: 5m                               # tiene que cumplirse 5 min seguidos
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.instance }} no responde"
          description: "El target {{ $labels.job }}/{{ $labels.instance }} lleva 5 minutos caído."

      - alert: DiscoCasiLleno
        expr: |
          (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}
           / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"}) * 100 < 10
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.instance }}: queda menos del 10% en {{ $labels.mountpoint }}"
```

```
   inactive ──(expr se cumple)──► pending ──(pasa el "for")──► firing ──► Alertmanager
```

### Alertmanager: agrupar y enrutar

```yaml
# /etc/alertmanager/alertmanager.yml
global:
  smtp_smarthost: "smtp.lab.local:587"
  smtp_from: "alertas@lab.local"

route:
  receiver: correo-devops                   # por defecto
  group_by: ["alertname", "instance"]       # una notificación por grupo, no por serie
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - matchers: ['severity="critical"']
      receiver: guardia

receivers:
  - name: correo-devops
    email_configs:
      - to: "devops@lab.local"
  - name: guardia
    telegram_configs:
      - bot_token_file: /etc/alertmanager/telegram_token
        chat_id: -1001234567890
```

| Concepto | Qué hace |
|---|---|
| **Agrupación** | 50 nodos caídos a la vez = 1 mensaje, no 50 |
| **Inhibición** | si salta "centro de datos caído", calla las alertas de cada servidor |
| **Silencio** | mute temporal durante una ventana de mantenimiento (desde la UI `:9093`) |

> Una alerta debe ser **accionable**: si al recibirla no hay nada que hacer, sobra y
> entrena al equipo a ignorar las alertas.

## Grafana

Plataforma de visualización de datos: una aplicación web donde construyes paneles
(*dashboards*) con gráficas, tablas y alertas a partir de datos que viven en otro sitio.

- **Puede conectarse a**: Prometheus, InfluxDB, Elasticsearch, PostgreSQL, Loki, etc.
- La comunidad publica **dashboards ya hechos** que importas por un ID
  (ej.: *Node Exporter Full*, ID **1860**).
- También hace **alertas**: por Telegram, Slack, correo, etc.

Puerto por defecto **3000**; usuario inicial `admin` / `admin` (obliga a cambiarla en el
primer login).

### Datasource como código (provisioning)

```yaml
# /etc/grafana/provisioning/datasources/prometheus.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus.lab.local:9090
    isDefault: true
```

Los dashboards también se pueden provisionar como JSON desde disco: así se versionan en Git
en vez de vivir solo en la base de datos de Grafana.

### Variables de dashboard

Permiten un desplegable para elegir servidor sin duplicar paneles:

```
Variable:  instance
Query:     label_values(node_uname_info, instance)

Panel:     100 - (avg(rate(node_cpu_seconds_total{mode="idle", instance=~"$instance"}[5m])) * 100)
```

(`=~` en vez de `=` para que funcione con selección múltiple y "All")

### ¿Alertas en Grafana o en Prometheus?

| | Prometheus + Alertmanager | Grafana Alerting |
|---|---|---|
| Definición | ficheros YAML (versionables) | UI o provisioning |
| Fuentes | solo Prometheus | cualquier datasource (Loki, SQL…) |
| Si Grafana cae | las alertas siguen | no hay alertas |

## En Kubernetes: kube-prometheus-stack

La chart de Helm que monta todo de una vez: Prometheus Operator, Prometheus, Alertmanager,
Grafana, node-exporter (DaemonSet), kube-state-metrics y reglas y dashboards predefinidos.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace
```

Con el Operator no se edita `prometheus.yml`: se crean recursos **ServiceMonitor** que dicen
qué Services raspar.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mi-app
  namespace: monitoring
  labels:
    release: monitoring          # ← debe coincidir con el selector del Prometheus de la chart
spec:
  namespaceSelector:
    matchNames: ["mi-namespace"]
  selector:
    matchLabels:
      app: mi-app                # labels del Service
  endpoints:
    - port: http-metrics         # NOMBRE del puerto en el Service, no el número
      path: /metrics
      interval: 30s
```

| Componente | Qué aporta |
|---|---|
| node-exporter | métricas de cada nodo |
| kube-state-metrics | estado de los objetos: réplicas deseadas vs disponibles, pods en `CrashLoopBackOff`… |
| cAdvisor (dentro del kubelet) | CPU y memoria por contenedor |
| PrometheusRule (CRD) | reglas de alerta como recurso de Kubernetes |

## Exporters habituales

| Exporter | Puerto | Mide |
|---|---|---|
| node_exporter | 9100 | Linux |
| windows_exporter | 9182 | Windows |
| blackbox_exporter | 9115 | sondas desde fuera: HTTP, TCP, ICMP, caducidad de certificados |
| postgres_exporter | 9187 | PostgreSQL |
| mysqld_exporter | 9104 | MySQL / MariaDB |
| redis_exporter | 9121 | Redis |
| kube-state-metrics | 8080 | objetos de Kubernetes |

> El **blackbox_exporter** es el que responde a "¿funciona desde el punto de vista del
> usuario?". Con `probe_ssl_earliest_cert_expiry` avisa antes de que caduque un certificado.

## Diagnóstico

| Síntoma | Causa habitual |
|---|---|
| Target `DOWN` con `connection refused` | el exporter no está arrancado o escucha solo en `127.0.0.1` |
| Target `DOWN` con `context deadline exceeded` | firewall (9100 cerrado) o el scrape tarda más que `scrape_timeout` |
| Target `DOWN` con `server returned HTTP status 401/403` | el endpoint pide autenticación |
| El ServiceMonitor no aparece en `/targets` | falta la etiqueta `release:` o el `port` no es el **nombre** del puerto del Service |
| Gráficas con huecos | reinicios de Prometheus, scrapes fallidos o ventana de `rate()` demasiado corta |
| `rate()` da valores absurdos | aplicado sobre un gauge (solo sirve para counters) |
| Prometheus consume muchísima RAM | explosión de cardinalidad: `topk(10, count by (__name__) ({__name__=~".+"}))` |
| "No data" en un dashboard importado | el nombre del datasource o de las etiquetas (`job`, `instance`) no coincide con el tuyo |
