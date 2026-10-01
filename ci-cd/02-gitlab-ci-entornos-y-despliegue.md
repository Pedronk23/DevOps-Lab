# CI/CD con GitLab (2) — Artefactos, entornos, plantillas y estrategias de despliegue

> Continuación de [CI/CD con GitLab](01-gitlab-ci.md): artefactos, runners, variables, registro
> de imágenes, entornos, plantillas, pipelines programadas y estrategias de despliegue.

## Artefactos (`artifacts`)

Un **artefacto** (*artifact*) es la lista de ficheros y directorios que genera un job
**al terminar** y que GitLab guarda para descargarlos o pasarlos a jobs posteriores.

```yaml
build:
  stage: build
  script:
    - mkdir -p dist
    - echo "build $CI_COMMIT_SHORT_SHA" > dist/version.txt
  artifacts:
    paths:
      - dist/
    expire_in: 1 week     # por defecto, 30 días
    when: on_success      # on_success | on_failure | always
```

- Por defecto, un job **descarga los artefactos de todos los jobs de stages anteriores**. Se
  puede limitar con `dependencies:` o `needs:`.
- `artifacts:reports:` sirve para informes que GitLab entiende y muestra en el pipeline o en la
  MR (`junit`, `codequality`, `coverage_report`…).

En la interfaz:

- En la vista de un job, el **panel derecho** permite **cambiar entre los jobs del pipeline** y
  **ver o descargar sus artefactos** (*Browse*, *Download*, *Keep*).
- **Build → Artifacts** lista todos los artefactos del proyecto.

### Artefactos vs caché

| | Artefactos (`artifacts`) | Caché (`cache`) |
|---|---|---|
| Para qué | Resultado del job (compilados, binarios, informes) | Acelerar jobs (dependencias: `node_modules`, caché de pip) |
| Entre stages | Sí, se pasan automáticamente | No garantizado |
| Descargable desde la interfaz | Sí | No |
| Fiabilidad | Garantizado | Sin garantía (*best effort*): puede no estar |

## Imagen Docker y stage por defecto

Cada job puede usar una imagen Docker distinta según lo que necesite (Python para las pruebas,
Node para el frontend, `docker` para construir imágenes, `alpine` para scripts ligeros…).

```yaml
default:
  image: python:3.12      # imagen para todos los jobs que no indiquen otra

test:
  script: pytest          # no indica stage → va a "test"

build_frontend:
  stage: build
  image: node:22          # sobrescribe la imagen por defecto
  script: npm run build
```

- Si un job no indica `stage:`, va al stage **`test`**.
- Si no hay `image:` ni en el job ni en `default:`, se usa la imagen por defecto del runner. En
  los runners alojados de GitLab.com es una imagen de **Ruby** (`ruby:3.1`).

## Runners

- Aplicación que **ejecuta los jobs** de GitLab CI/CD: recoge los jobs pendientes, los ejecuta
  y devuelve el resultado y el log.
- De **código abierto** y escrita en **Go** (un único binario: `gitlab-runner`).
- Se pueden **añadir o quitar** runners en cualquier momento.
- GitLab.com ofrece varios **runners compartidos** listos para usar (con minutos de cómputo
  limitados en el plan gratuito).
- Se pueden instalar en **infraestructura propia** (máquina virtual, servidor físico,
  Kubernetes…).

Tipos según su alcance:

| Tipo | Disponible para |
|---|---|
| Instancia (compartido) | Todos los proyectos de la instancia |
| Grupo | Todos los proyectos de un grupo |
| Proyecto | Un solo proyecto |

Con `tags:` eliges qué runner coge cada job:

```yaml
deploy:
  tags: [linux, docker]
  script: ./deploy.sh
```

Registro de un runner propio: se crea en **Settings → CI/CD → Runners**, GitLab da un token
(`glrt-...`) y en la máquina se ejecuta:

```bash
gitlab-runner register --url https://gitlab.com --token glrt-XXXX
```

> **Buena práctica**: instalar el runner en una **máquina distinta** de la instancia de GitLab.
> Los jobs ejecutan código arbitrario (riesgo de seguridad) y consumen la CPU y la RAM que
> necesita el servidor.

## Operadores `&&` y `||` en los scripts

En GitLab, si cualquier comando del `script` devuelve un código de salida distinto de 0,
**el job falla**. Estos operadores de shell permiten encadenar comandos y controlar los fallos:

| Operador | Nombre | Comportamiento |
|---|---|---|
| `a && b` | Y lógico (AND) | `b` solo se ejecuta si `a` **tiene éxito** |
| `a \|\| b` | O lógico (OR) | `b` solo se ejecuta si `a` **falla** |
| `a ; b` | Secuencia | `b` se ejecuta siempre |
| `a \| b` | Tubería (*pipe*) | La salida de `a` es la entrada de `b` |

```bash
cd /tmp || mkdir /tmp                          # si no puede entrar en /tmp, lo crea
pip install -r requirements.txt && pytest      # pruebas solo si la instalación va bien
rm fichero_temporal || true                    # aunque falle, el job sigue
```

> Para crear directorios es más limpio `mkdir -p dir`, que no falla si ya existe.

## Variables CI/CD y secretos

Guardan información para usarla en los jobs como `$VARIABLE`.

> Aunque en el YAML escribas un número, **todas las variables son texto** (variables de
> entorno). Si necesitas operar con un número, conviértelo en el script.

Dónde se definen:

- En el `.gitlab-ci.yml` (global o por job) → solo valores **no secretos**.
- En **Settings → CI/CD → Variables** → **secretos** (tokens, contraseñas, claves de API). Nunca
  en el repo.
- GitLab añade además **variables predefinidas**: `$CI_COMMIT_SHA`, `$CI_COMMIT_BRANCH`,
  `$CI_DEFAULT_BRANCH`, `$CI_REGISTRY_IMAGE`…

```yaml
variables:
  PYTHON_VERSION: "3.12"

test:
  image: python:$PYTHON_VERSION
  script:
    - echo "Rama: $CI_COMMIT_BRANCH"
```

Opciones al crear una variable en la interfaz:

| Opción | Qué hace |
|---|---|
| **Protect** | Solo disponible en pipelines de ramas o etiquetas **protegidas** (p. ej. `main`) |
| **Mask** | Se oculta en los logs como `[MASKED]`. Requiere al menos 8 caracteres y una sola línea |
| **Masked and hidden** | Además, el valor no se puede volver a ver en la interfaz una vez guardado |
| **Type: File** | GitLab escribe el valor en un fichero temporal y la variable contiene la **ruta**. Útil para claves SSH, kubeconfig o certificados |
| **Environment scope** | Limita la variable a un entorno (`production`, `review/*`…) |

## Instalación propia vs nube, y GitLab Pages

| Modalidad | Descripción |
|---|---|
| **GitLab.com** (nube, SaaS) | Lo gestiona GitLab. Sin mantenimiento y con runners compartidos incluidos |
| **Self-managed** (instalación propia, *on-premise*) | Lo instalas tú (servidor propio, máquina virtual, Kubernetes). Control total de datos, versiones y red; las actualizaciones y copias de seguridad son cosa tuya |
| **GitLab Dedicated** | Instancia exclusiva gestionada por GitLab, para empresas con requisitos de cumplimiento normativo |

**GitLab Pages**: alojamiento gratuito de webs **estáticas** desde un repo. Se publica con un
job llamado `pages` que deja la web en `public/` como artefacto:

```yaml
pages:
  stage: deploy
  script:
    - mkdir -p public
    - cp -r site/* public/
  artifacts:
    paths:
      - public
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

La web queda en `https://<usuario-o-grupo>.gitlab.io/<proyecto>`.

## Flujo de trabajo

**Primer principio de CI/CD: nunca trabajar directamente en `main`.**

```
feature/nueva-funcion ─► Merge Request ─► pipeline (lint + pruebas) ─► revisión ─► fusión en main ─► despliegue
```

- Cada cambio va en su propia rama `feature/...`.
- Se abre una **Merge Request** (MR) y el pipeline valida el cambio antes de fusionarlo.
- Protege `main` en **Settings → Repository → Protected branches** (nada de *push* directo,
  solo fusiones mediante MR).
- Puedes exigir que el pipeline pase antes de fusionar:
  **Settings → Merge requests → Pipelines must succeed**.

## Proyecto de ejemplo: web en Python

Pasos del curso: crear la web en Python → lint y pruebas → guardar los resultados como
artefactos → construir la imagen Docker → subirla al registro → desplegar.

### Lint y pruebas

Un **linter** (analizador estático) revisa el código **sin ejecutarlo** y detecta errores de
estilo y fallos comunes.

- **flake8**: linter de Python (estilo PEP 8, imports sin usar, variables no definidas…).
- **ansible-lint**: linter de playbooks y roles de Ansible.

```yaml
lint_python:
  stage: test
  image: python:3.12
  script:
    - pip install flake8
    - flake8 --tee --output-file=flake8-report.txt .
  artifacts:
    when: always
    paths:
      - flake8-report.txt
    expire_in: 1 week

lint_ansible:
  stage: test
  image: python:3.12
  script:
    - pip install ansible-lint
    - ansible-lint -f codeclimate > ansible-lint.json
  artifacts:
    when: always
    reports:
      codequality: ansible-lint.json

test:
  stage: test
  image: python:3.12
  script:
    - pip install -r requirements.txt
    - pytest --junitxml=report.xml
  artifacts:
    when: always
    reports:
      junit: report.xml
```

- `--tee` muestra el resultado en el log **y** lo guarda en el fichero.
- `when: always` sube el artefacto aunque el job falle, que es justo cuando quieres verlo.
- El informe `junit` aparece en la pestaña **Tests** del pipeline; el de `codequality`, en la
  Merge Request.

### Heroku, Gunicorn y Procfile

- **Heroku**: PaaS (*Platform as a Service*, plataforma como servicio). Subes el código o una
  imagen y Heroku lo ejecuta sin que gestiones servidores. Cada aplicación corre en
  contenedores llamados **dynos**.
- **Gunicorn**: servidor WSGI para Python. El servidor de desarrollo de Flask o Django no sirve
  para producción; Gunicorn sí (varios procesos de trabajo, gestión de procesos).
- **Procfile**: fichero en la raíz del repo que indica a Heroku **qué comando ejecutar** para
  cada **tipo de proceso**:

```
web: gunicorn app:app
worker: python worker.py
```

- `web` es el único tipo de proceso que recibe tráfico HTTP.
- `app:app` = módulo `app.py`, objeto `app` (la aplicación Flask).
- Desde CI, la CLI de Heroku se autentica con la variable `HEROKU_API_KEY`, guardada como
  variable **masked** en Settings → CI/CD.

> Heroku eliminó su plan gratuito en noviembre de 2022. Para practicar se puede usar otra PaaS
> o desplegar en un servidor propio o en Kubernetes.

### Construir la imagen y registro de contenedores de GitLab

GitLab incluye un **registro de imágenes Docker** (*Container Registry*) en cada proyecto,
gratis en todos los planes (en GitLab.com el espacio cuenta para la cuota de almacenamiento).
Se ve en **Deploy → Container Registry**.

Formato del nombre de imagen:

```
<registro>/<usuario-o-grupo>/<proyecto>/<imagen>:<etiqueta>
registry.gitlab.com/mi-usuario/mi-web/app:1.0.0
```

La parte `/<imagen>` es opcional: `registry.gitlab.com/mi-usuario/mi-web:1.0.0` también es
válido. GitLab da variables predefinidas para no escribir nada a mano:

| Variable | Valor |
|---|---|
| `$CI_REGISTRY` | `registry.gitlab.com` |
| `$CI_REGISTRY_IMAGE` | `registry.gitlab.com/<usuario-o-grupo>/<proyecto>` |
| `$CI_REGISTRY_USER` / `$CI_REGISTRY_PASSWORD` | Credenciales temporales del job |

Job de construcción añadido al `.gitlab-ci.yml`:

```yaml
build_image:
  stage: build
  image: docker:27
  services:
    - docker:27-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
  script:
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA" -t "$CI_REGISTRY_IMAGE:latest" .
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"
    - docker push "$CI_REGISTRY_IMAGE:latest"
```

- `docker:27-dind` (*Docker-in-Docker*) levanta un demonio Docker como servicio para poder
  construir imágenes dentro del job.
- Etiquetar con el SHA del commit permite saber qué código lleva cada imagen y volver a una
  versión anterior.

### `docker run`

```
docker run [opciones] <imagen> [comando]
```

| Parte | Significado |
|---|---|
| `run` | Crea un contenedor nuevo a partir de la imagen y lo arranca |
| `<imagen>` | Imagen que se usa |
| `[comando]` | Opcional. **Sobrescribe el comando por defecto** (`CMD`) de la imagen |

```bash
docker run python:3.12 python --version            # ejecuta un comando y termina
docker run -d -p 8000:8000 --name web mi-web:1.0   # en segundo plano, publicando el puerto 8000
docker run --rm -it alpine sh                      # shell interactiva; --rm borra el contenedor al salir
```

En GitLab, el `image:` + `script:` de un job equivale en la práctica a un `docker run` de esa
imagen ejecutando tus comandos.

## Entornos (`environment`)

Un **entorno** (*environment*) representa dónde se despliega el código (`staging`,
`production`…). GitLab guarda el historial de despliegues de cada uno en
**Operate → Environments**.

**Producción va aparte**: crea una aplicación (o infraestructura) propia para producción,
independiente de desarrollo y preproducción. Así un fallo en preproducción no afecta a los
usuarios y cada entorno tiene sus propias variables y secretos.

### `only` / `rules`: cuándo se ejecuta un job

`only:` controla cuándo debe ejecutarse un job:

```yaml
deploy_production:
  only:
    - main
```

> `only` / `except` ya no se desarrollan activamente; GitLab recomienda **`rules:`**, que es
> más flexible:

```yaml
deploy_production:
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

### Despliegue manual a producción

```yaml
stages: [test, build, deploy, production]

deploy_staging:
  stage: deploy
  script: ./deploy.sh staging
  environment:
    name: staging
    url: https://staging.example.com
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

deploy_production:
  stage: production
  script: ./deploy.sh production
  environment:
    name: production
    url: https://example.com
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual        # aparece un botón ▶ en el pipeline para lanzarlo
```

Así se pasa de despliegue continuo (*Continuous Deployment*) a entrega continua
(*Continuous Delivery*): todo llega solo a `staging`, pero producción necesita que alguien
pulse el botón.

### Entornos estáticos y dinámicos

| Tipo | Nombre | Ejemplo |
|---|---|---|
| **Estático** | Fijo, siempre existe | `staging`, `production` |
| **Dinámico** | Se crea por rama o MR usando variables | `review/$CI_COMMIT_REF_SLUG` |

Los entornos dinámicos (aplicaciones de revisión o *review apps*) despliegan cada rama de
funcionalidad en su propia URL para probarla antes de fusionarla.

### Vuelta atrás (rollback)

En **Operate → Environments → \<entorno\>** aparece el historial de despliegues. Desde ahí se
puede **volver a desplegar una versión anterior** (*Rollback* / *Re-deploy*), que relanza el
job de despliegue de ese commit.

Por eso conviene que cada despliegue use una imagen con una etiqueta inmutable (el SHA del
commit) y no `latest`.

### Entornos dinámicos: crear y borrar automáticamente

En un equipo se pueden crear muchísimos entornos dinámicos (uno por rama), así que hay que
**automatizar su borrado**:

```yaml
deploy_feature:
  stage: deploy
  before_script:
    - export FEATURE_APP="$CI_ENVIRONMENT_SLUG"
  script:
    - ./deploy.sh "$FEATURE_APP"          # crea/actualiza la app de esta rama
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_ENVIRONMENT_SLUG.example.com
    on_stop: stop_feature
    auto_stop_in: 1 week
  rules:
    - if: $CI_COMMIT_BRANCH && $CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH

stop_feature:
  stage: deploy
  before_script:
    - export FEATURE_APP="$CI_ENVIRONMENT_SLUG"
  script:
    # en el curso, con Heroku:
    - heroku apps:destroy --app "$FEATURE_APP" --confirm "$FEATURE_APP"
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
  rules:
    - if: $CI_COMMIT_BRANCH && $CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH
      when: manual
```

- `on_stop`: job que destruye el entorno. Se puede lanzar a mano, y GitLab lo ejecuta solo
  cuando se borra la rama (p. ej. al fusionar la MR).
- `action: stop`: marca ese job como el que para el entorno.
- `auto_stop_in`: GitLab para el entorno automáticamente pasado ese tiempo.
- `--confirm <app>` evita la pregunta interactiva de Heroku, que bloquearía el job.

**Problema con variables que contienen variables**: definir en `variables:` algo como
`FEATURE_APP: $CI_ENVIRONMENT_SLUG` puede dar problemas, porque la variable interna no siempre
se expande en ese punto. **Solución**: hacer el `export` en `before_script`, donde todas las
variables del job ya están disponibles.

## Plantillas de jobs (anclas YAML)

Sirven para no repetir la misma configuración en varios jobs.

- **Job oculto**: si el nombre de un job empieza por `.`, **no se ejecuta**. Sirve como
  plantilla o para desactivar un job temporalmente.
- **`&nombre`** (ancla, *anchor*): marca un bloque como plantilla.
- **`<<: *nombre`** (clave de fusión + alias, *merge key* + *alias*): copia el contenido de la
  plantilla en otro job.

```yaml
.deploy_template: &deploy
  image: alpine:3.20
  before_script:
    - apk add --no-cache curl
  script:
    - ./deploy.sh "$ENV"

deploy_staging:
  <<: *deploy
  variables:
    ENV: staging

deploy_production:
  <<: *deploy
  variables:
    ENV: production
  when: manual
```

Las claves que pongas en el job **sobrescriben** las de la plantilla.

### Alternativa propia de GitLab: `extends`

```yaml
.deploy:
  image: alpine:3.20
  script:
    - ./deploy.sh "$ENV"

deploy_staging:
  extends: .deploy
  variables:
    ENV: staging
```

| | Anclas (`&` / `<<: *`) | `extends` |
|---|---|---|
| Origen | Estándar YAML | Propio de GitLab |
| Entre ficheros (`include:`) | No, solo en el mismo fichero | Sí |
| Fusión | Superficial (sustituye claves completas) | Profunda (fusiona diccionarios) |
| Legibilidad | Menor | Mayor |

Para reutilizar solo una sección concreta de otro job existe `!reference [.deploy, script]`.

## Validar el YAML (CI Lint)

Valida la sintaxis del `.gitlab-ci.yml` **antes** de hacer *push*:

- Interfaz: **Build → Pipeline editor** (pestaña *Validate*) o la ruta `/-/ci/lint` del
  proyecto.
- Línea de comandos: `glab ci lint`.

Otros linters útiles dentro del pipeline: `yamllint` (YAML), `hadolint` (Dockerfile),
`shellcheck` (bash), `flake8` (Python), `ansible-lint` (Ansible).

## Pipelines programadas

**Build → Pipeline schedules → New schedule**. Se configura:

- **Cuándo**: hora, día, mes… con sintaxis **cron**, y la zona horaria.
- **Rama objetivo** (*Target branch*): rama sobre la que corre el pipeline.
- **Variables** propias de esa programación.

Uso típico: compilaciones nocturnas, escaneos de seguridad, limpiezas, pruebas largas.

```yaml
nightly_tests:
  script: pytest tests/integration
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
```

### Sintaxis cron

```
┌──────── minuto (0-59)
│ ┌────── hora (0-23)
│ │ ┌──── día del mes (1-31)
│ │ │ ┌── mes (1-12)
│ │ │ │ ┌ día de la semana (0-6, 0 = domingo)
│ │ │ │ │
* * * * *
```

| Expresión | Significado |
|---|---|
| `0 2 * * *` | Todos los días a las 02:00 |
| `0 9 * * 1-5` | De lunes a viernes a las 09:00 |
| `*/15 * * * *` | Cada 15 minutos |
| `0 0 1 * *` | El día 1 de cada mes a medianoche |

[crontab.guru](https://crontab.guru) traduce cualquier expresión cron a lenguaje natural (en
inglés). Muy útil para comprobarlas.

## Tiempo máximo (`timeout`)

Tiempo máximo que puede ejecutarse un job. Si lo supera, GitLab lo cancela y lo marca como
fallido.

| Nivel | Dónde | Notas |
|---|---|---|
| **Job** | `timeout:` en el job | Puede superar al del proyecto, pero no al del runner |
| **Runner** | Ajustes del runner (*Maximum job timeout*) | Límite superior para todos los jobs que ejecute ese runner |
| **Proyecto** | Settings → CI/CD → General pipelines → Timeout | Por defecto, **60 minutos** |

```yaml
build:
  script: make build
  timeout: 1h 30m
```

## Estrategias de despliegue

### Recreación (*Recreate*)

Se paran todas las instancias de la versión antigua y se arrancan las de la nueva.

```
v1 v1 v1  ─►  (servicio caído)  ─►  v2 v2 v2
```

**Ventajas**
- Muy simple de implementar.
- Nunca conviven dos versiones (útil si hay cambios incompatibles en la base de datos).
- No necesita recursos extra.

**Inconvenientes**
- **Hay caída de servicio** (*downtime*) durante el cambio.
- Vuelta atrás lenta: hay que volver a desplegar la versión anterior.

### Azul-verde (*Blue-Green*)

Dos entornos idénticos: **azul** (*blue*, versión actual, recibe el tráfico) y **verde**
(*green*, versión nueva). Se despliega en verde, se prueba y se cambia todo el tráfico de golpe.

```
                         ┌─► AZUL  (v1)  ← tráfico actual
Usuarios ─► balanceador ─┤
                         └─► VERDE (v2)  ← se prueba; luego el balanceador apunta aquí
```

**Ventajas**
- Sin caída de servicio.
- Vuelta atrás instantánea: basta con volver a apuntar a azul.
- Se prueba la versión nueva en un entorno real antes de darle tráfico.

**Inconvenientes**
- Infraestructura doble (más coste).
- Si comparten base de datos, los cambios de esquema deben ser compatibles con ambas versiones.
- El cambio afecta a todos los usuarios a la vez.

### Canario (*Canary*)

Se despliega la versión nueva para un **porcentaje pequeño** del tráfico (p. ej. el 10 %), se
observa cómo reacciona (errores, latencia, consumo) y se va aumentando hasta el 100 %.

```
v1 ██████████████████   90 %
v2 ██                   10 %  ─► 25 % ─► 50 % ─► 100 %
```

**Ventajas**
- Riesgo limitado: un fallo solo afecta a una parte de los usuarios.
- Se valida con tráfico real.
- Vuelta atrás rápida: devolver el porcentaje a 0.

**Inconvenientes**
- Más complejo: necesita repartir el tráfico (balanceador, malla de servicios o
  *service mesh*, Argo Rollouts, Flagger…) y buena monitorización.
- Conviven dos versiones durante un tiempo.
- Despliegue más lento.

### Pruebas A/B (*A/B testing*)

Parecido al canario, pero el objetivo **es de negocio, no técnico**: se muestran dos versiones
a grupos de usuarios elegidos por criterios (país, dispositivo, cookie…) para medir cuál
funciona mejor (conversión, clics, tiempo de uso…).

**Ventajas**
- Decisiones basadas en datos reales de uso.
- Segmentación precisa de usuarios.

**Inconvenientes**
- Necesita enrutado por usuario (interruptores de funcionalidad o *feature flags*, cookies) y
  herramientas de analítica.
- Hace falta suficiente tráfico para que los resultados sean fiables.
- Más complejidad en el código, con varias variantes vivas a la vez.

### Resumen

| Estrategia | Caída de servicio | Coste de infraestructura | Vuelta atrás | Riesgo | Complejidad |
|---|---|---|---|---|---|
| Recreación | Sí | Bajo | Lenta | Alto | Baja |
| Azul-verde | No | Alto (x2) | Instantánea | Medio | Media |
| Canario | No | Bajo-medio | Rápida | Bajo | Alta |
| Pruebas A/B | No | Medio | Rápida | Bajo | Alta |
