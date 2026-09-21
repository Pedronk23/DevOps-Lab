# CI/CD con GitLab

## Conceptos

- **SDLC** (*Software Development Life Cycle*): ciclo de vida del desarrollo de software.
- **DevOps** (*Development + Operations*): unir desarrollo y operaciones para entregar
  software de forma rápida, automatizada y fiable.
- **CI** (*Continuous Integration*): cada cambio se integra en la rama principal y se valida
  automáticamente (build + tests).
- **CD** puede significar dos cosas:
  - **Continuous Delivery**: el artefacto queda siempre listo para desplegar, pero el paso a
    producción requiere **aprobación manual**.
  - **Continuous Deployment**: se despliega a producción automáticamente, **sin aprobación
    manual**.

```
   Continuous Delivery                 Continuous Deployment
   ───────────────────                 ─────────────────────
   commit → build → test →             commit → build → test →
   staging → [botón] → producción      staging → producción (automático)
```

## Pipeline

```
START ─► TEST ─► BUILD ─► DELIVER ─► RELEASE ─► END
```

Un **pipeline** se compone de **stages** (etapas, se ejecutan en orden) y cada stage contiene
**jobs** (se ejecutan en paralelo dentro del stage).

## GitLab

Plataforma DevOps completa: repositorio, issues, CI/CD, registro de contenedores, gestión de
secretos…

El pipeline se define en `.gitlab-ci.yml` en la raíz del repo y lo ejecutan los
**GitLab Runners**. Si el fichero existe, **cada push dispara un pipeline**.

```yaml
stages:
  - test
  - build
  - deliver
  - release

variables:
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

test:
  stage: test
  image: python:3.12
  script:
    - pip install -r requirements.txt
    - pytest

build:
  stage: build
  image: docker:27
  services: [docker:27-dind]
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $IMAGE .
    - docker push $IMAGE

deliver:
  stage: deliver
  script:
    - echo "Desplegando $IMAGE en staging"
  environment: staging

release:
  stage: release
  script:
    - echo "Desplegando $IMAGE en producción"
  environment: production
  when: manual          # Continuous Delivery; quitarlo = Continuous Deployment
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

En la interfaz (**Build → Pipelines**) se ve cada pipeline con sus stages y jobs, y se puede
entrar en cada job para leer sus logs.

## Stages y jobs

- **Stages**: fases del pipeline. Definen el **orden de ejecución**.
- **Job**: unidad de trabajo. Indica a qué stage pertenece con `stage:`.

```yaml
build-job:
  stage: build
  script: npm ci

test-job:
  stage: test
  script: npm test

deploy-job:
  stage: deploy
  script: ./deploy.sh
```

Si no declaras `stages:`, GitLab usa por defecto:

| Stage | Notas |
|---|---|
| `.pre` | siempre se ejecuta **la primera** (reservada) |
| `build` | |
| `test` | stage por defecto si un job no indica `stage:` |
| `deploy` | |
| `.post` | siempre se ejecuta **la última** (reservada) |

Reglas de ejecución:

- Los jobs del **mismo stage** corren **en paralelo** (si hay runners libres).
- Un stage no empieza hasta que **terminan con éxito** todos los jobs del anterior.
- Si un job falla, por defecto el pipeline se detiene (salvo `allow_failure: true`).

## Runners, executors y logs

Al entrar en un job se ve su log. Todas las ejecuciones siguen las mismas fases:

```
Running with gitlab-runner 17.x.x                      → versión del runner que coge el job
Preparing the "docker" executor                        → prepara el executor (aquí Docker)
Preparing environment                                  → arranca el contenedor con la imagen del job
Getting source from Git repository                     → clona o actualiza el repo en el contenedor
Executing "step_script" stage of the job script        → ejecuta before_script + script
Cleaning up project directory and file based variables → limpia las variables de tipo fichero
Job succeeded
```

- **Runner**: agente que ejecuta los jobs. Puede ser compartido (de GitLab) o propio
  (*self-hosted*).
- **Executor**: la forma en que el runner ejecuta cada job.

| Executor | Cómo corre el job |
|---|---|
| `docker` | un contenedor limpio por job (el más común); la imagen la marca `image:` |
| `shell` | directamente en la máquina del runner, sin aislamiento |
| `kubernetes` | un pod por job |

> Cada job arranca en un entorno limpio: lo que un job deja en disco **no** lo ve el
> siguiente, salvo que se pase con `artifacts` o `cache`.

## Variables y secretos

| Variable predefinida | Contenido |
|---|---|
| `$CI_COMMIT_BRANCH` | rama del commit |
| `$CI_DEFAULT_BRANCH` | rama por defecto del proyecto (`main`) |
| `$CI_COMMIT_SHORT_SHA` | identificador corto del commit: buen tag de imagen |
| `$CI_REGISTRY`, `$CI_REGISTRY_IMAGE` | registro de contenedores del proyecto |
| `$CI_PIPELINE_SOURCE` | qué disparó el pipeline (`push`, `merge_request_event`, `schedule`…) |

Los secretos (tokens, contraseñas, kubeconfig) se declaran en
**Settings → CI/CD → Variables**, nunca en el `.gitlab-ci.yml`:

- **Masked**: se ocultan en los logs.
- **Protected**: solo se exponen en ramas y etiquetas protegidas.
- Tipo **File**: se montan como fichero, útil para un kubeconfig o una clave.

> Un `echo $TOKEN` en un script lo saca por el log. Con *masked* aparece como `[MASKED]`,
> pero lo sano es no imprimirlo.

## Node.js y npm

Aparecen mucho en los ejemplos de pipelines.

- **Node.js**: entorno de ejecución de JavaScript **fuera del navegador** (motor V8). Muy
  usado en aplicaciones en tiempo real (chats, APIs, websockets) por su modelo asíncrono
  basado en eventos.
- **npm** (*Node Package Manager*): instala librerías y plugins de terceros. Las dependencias
  se declaran en `package.json` y las versiones exactas quedan fijadas en `package-lock.json`.

```bash
npm install express    # añade una dependencia al proyecto
npm ci                 # instalación limpia desde package-lock.json (la recomendada en CI)
npm test               # ejecuta el script "test" de package.json
npm run build          # ejecuta el script "build"
```

Pipeline típico para una app Node:

```yaml
image: node:22

stages:
  - build
  - test
  - deploy

cache:
  key:
    files: [package-lock.json]
  paths: [node_modules/]

build:
  stage: build
  script:
    - npm ci
    - npm run build
  artifacts:
    paths: [dist/]

test:
  stage: test
  script:
    - npm ci
    - npm test

deploy:
  stage: deploy
  script:
    - echo "Desplegando dist/"
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

| Clave | Para qué |
|---|---|
| `cache` | reutiliza `node_modules` entre pipelines para no descargarlo todo cada vez |
| `artifacts` | pasa ficheros generados (`dist/`) a los jobs de stages posteriores |
| `rules` | decide si el job se crea, y con qué condiciones |
| `needs` | rompe el orden de stages: un job arranca en cuanto tiene lo que necesita |

## Encaje con el resto del repo

| Pieza | Nota |
|---|---|
| Ramas y versiones sobre las que corre el pipeline | [Git Flow](../git/gitflow.md) |
| Construcción de la imagen del job de build | [Docker: fundamentos e imágenes](../docker/01-fundamentos-y-dockerfile.md) |
| Publicación de artefactos e imágenes | [Transferencia y repositorios de artefactos](../redes/03-transferencia-y-artefactos.md) |
| Despliegue del resultado en un clúster | [Helm y Operators](../kubernetes/14-helm-y-operators.md) |
| Tests de roles de Ansible dentro del pipeline | [Molecule](../herramientas/molecule.md) |

## Errores típicos

| Síntoma | Causa habitual |
|---|---|
| `This job is stuck` | no hay runner disponible con esas *tags*, o están todos ocupados |
| `Cannot connect to the Docker daemon` en el job de build | falta el servicio `docker:dind` o el runner no es privilegiado |
| El pipeline no arranca | `.gitlab-ci.yml` no está en la raíz, o sus `rules` excluyen esa rama |
| `yaml invalid` | validarlo antes en **CI/CD → Editor** |
| Un secreto aparece en el log | variable sin *masked*, o impresa a propósito con `echo` |
| El job de despliegue corre en cada rama | falta `rules: - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH` |
