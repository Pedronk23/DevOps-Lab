# Git Flow

```
main     ──●(v0.1)───────────────────●(v0.2)──────────────────●(v1.0)
            │  ▲                        ▲                    ▲  │
hotfix      └──●────────────────────────┘                    │  │
               │                                             │  │
release        │                               ●─────────────●  │
               │                               ▲                │
develop     ───●────●──────●──────────●─────────●────────────────●
               │    ▲      ▲          ▲
feature        │    └──●───●──────────┘
               │
feature        └──●────●────●────●
```

## Ramas

- **main**: es la rama sagrada. Solo contiene código que está o estuvo en producción.
  Cada commit es una versión entregada (v0.1, v0.2…).
- **develop**: el tronco de integración. Aquí converge todo el trabajo del equipo; es el
  estado más reciente del código.
- **feature**: una rama por cada nueva funcionalidad. Sale de `develop` y regresa a `develop`
  con un merge cuando el trabajo está listo.
- **release**: zona de preparación antes de publicar. Aquí solo van correcciones menores y
  ajustes de última hora, no features nuevas.
- **hotfix**: para emergencias en producción. Sale directamente de `main`, se arregla el bug
  y se mergea a `main` y a `develop` (v1.1.1).

| Rama | Sale de | Vuelve a | Vida | Nombre típico |
|---|---|---|---|---|
| `main` | — | — | permanente | `main` |
| `develop` | `main` (una vez) | — | permanente | `develop` |
| `feature` | `develop` | `develop` | días | `feature/PROJ-123-login` |
| `release` | `develop` | `main` **y** `develop` | días | `release/1.2.0` |
| `hotfix` | `main` | `main` **y** `develop` | horas | `hotfix/1.2.1` |

## Flujo habitual

> Trabajas en **feature** → mergeas a **develop** → cuando hay suficiente, abres **release**
> → mergeas a **main** y **develop**.
> Y si hay un bug crítico en producción, **hotfix** te permite actuar sin esperar al próximo
> ciclo de release.

## Paso a paso con git

### Feature

```bash
git switch develop
git pull
git switch -c feature/PROJ-123-login

# … trabajo y commits …
git push -u origin feature/PROJ-123-login
```

Después, **Merge Request / Pull Request** hacia `develop` (revisión + pipeline en verde).
Si se hace a mano:

```bash
git switch develop
git pull
git merge --no-ff feature/PROJ-123-login
git push
git branch -d feature/PROJ-123-login
git push origin --delete feature/PROJ-123-login
```

### Release

```bash
git switch -c release/1.2.0 develop
# subir la versión (package.json, pom.xml, galaxy.yml, CHANGELOG…) y solo correcciones
git commit -am "chore: versión 1.2.0"

# publicar
git switch main
git merge --no-ff release/1.2.0
git tag -a v1.2.0 -m "Release 1.2.0"

# devolver los arreglos de la release a develop
git switch develop
git merge --no-ff release/1.2.0

git push origin main develop --tags
git branch -d release/1.2.0
```

### Hotfix

```bash
git switch -c hotfix/1.2.1 main
# … arreglo …
git commit -am "fix: timeout en la conexión a la BBDD"

git switch main
git merge --no-ff hotfix/1.2.1
git tag -a v1.2.1 -m "Hotfix 1.2.1"

git switch develop
git merge --no-ff hotfix/1.2.1        # si hay una release/ abierta, se mergea en esa en su lugar

git push origin main develop --tags
git branch -d hotfix/1.2.1
```

> Si el hotfix no se lleva a `develop`, **el bug vuelve** en la siguiente release.

## Por qué `--no-ff`

```
   merge fast-forward (por defecto si se puede)     merge --no-ff
   ────────────────────────────────────────────     ─────────────
   develop ──●──●──●──●──●                           develop ──●───────────●  ← commit de merge
                                                                \         /
   (la feature "desaparece": no se ve                            ●──●──●
    qué commits fueron juntos)                                   feature/login
```

Con `--no-ff` el historial conserva **qué commits formaban cada feature**, y revertir la
feature entera es revertir un solo commit de merge.

## Versionado semántico

Las etiquetas de `main` siguen **SemVer**: `MAYOR.MENOR.PARCHE`.

| Cambio | Sube | Ejemplo | Rama típica |
|---|---|---|---|
| Rompe compatibilidad | MAYOR | `1.4.2` → `2.0.0` | release |
| Funcionalidad nueva compatible | MENOR | `1.4.2` → `1.5.0` | release |
| Corrección de errores | PARCHE | `1.4.2` → `1.4.3` | hotfix |
| Pre-release | sufijo | `2.0.0-rc.1` | release |

```bash
git tag                                # listar
git tag -a v1.2.0 -m "Release 1.2.0"   # etiqueta anotada (autor, fecha, mensaje): la buena
git push origin v1.2.0                 # las etiquetas NO se suben con un push normal
git describe --tags                    # en qué versión estoy (v1.2.0-3-g8f80e10 = 3 commits después)
```

## Con la extensión git-flow

Automatiza los pasos anteriores:

```bash
git flow init -d                          # -d: nombres de rama por defecto
git flow feature start PROJ-123-login
git flow feature finish PROJ-123-login
git flow release start 1.2.0
git flow release finish 1.2.0             # merge a main y develop + tag
git flow hotfix start 1.2.1
git flow hotfix finish 1.2.1
```

> La extensión original lleva años sin mantenimiento (existen forks). Además, con Merge
> Requests obligatorios los `finish` locales no encajan bien. Lo importante es entender el
> flujo con git a secas.

## Commits convencionales

Formato habitual de mensajes, que además permite generar el CHANGELOG y calcular la versión
automáticamente:

```
<tipo>(<ámbito opcional>): <descripción>

feat(login): añadir autenticación con LDAP
fix(api): corregir timeout en /health
docs: ampliar nota de Git Flow
chore: actualizar dependencias
refactor(roles): extraer tareas comunes de nginx
feat!: eliminar soporte de Windows Server 2012     ← el "!" marca cambio incompatible
```

| Tipo | Afecta a la versión |
|---|---|
| `feat` | MENOR |
| `fix` | PARCHE |
| `feat!` / `BREAKING CHANGE:` | MAYOR |
| `docs`, `chore`, `refactor`, `test`, `ci`, `style` | no |

## Estrategias de merge en la MR/PR

| Estrategia | Resultado | Cuándo |
|---|---|---|
| **Merge commit** | conserva todos los commits + un commit de merge | Git Flow clásico, historial fiel |
| **Squash** | toda la rama se convierte en **un** commit | features con commits "wip", "arreglo", "otro arreglo" |
| **Rebase / fast-forward** | commits reaplicados en línea, sin merge | historial lineal; exige ramas al día |

## Alternativas a Git Flow

```
   GIT FLOW                  GITHUB FLOW               TRUNK-BASED
   main + develop +          main + ramas cortas       main + ramas de horas
   feature/release/hotfix    que se despliegan         (o commits directos)
                             al mergear                + feature flags

   releases planificadas     despliegue continuo       despliegue continuo,
   y versionadas                                       equipos maduros en CI
```

| Modelo | Encaja con | No encaja con |
|---|---|---|
| **Git Flow** | software con versiones (librerías, colecciones de Ansible, apps instaladas en cliente), varias versiones en soporte | despliegue continuo varias veces al día |
| **GitHub Flow** | webs y servicios que se despliegan al mergear | mantener varias versiones a la vez |
| **GitLab Flow** | GitHub Flow + ramas por entorno (`staging`, `production`) o ramas de release | — |
| **Trunk-based** | CI/CD madura, tests automáticos sólidos, feature flags | equipos sin tests automáticos |

> El propio autor de Git Flow (Vincent Driessen) añadió en 2020 una nota a su artículo: para
> aplicaciones web con entrega continua recomienda algo más simple, como GitHub Flow.

## Proteger las ramas

En GitLab/GitHub, para `main` y `develop`:

- Nadie hace `push` directo: **solo por Merge Request**.
- Pipeline en verde obligatorio antes de mergear.
- Al menos una aprobación.
- Prohibido el `push --force`.
- Etiquetas `v*` protegidas: solo quien publica releases puede crearlas.

## Chuleta para el día a día

| Tarea | Comando |
|---|---|
| Ver estado | `git status -sb` |
| Historial en árbol | `git log --oneline --graph --all --decorate` |
| Ramas con su remota y si van por delante/detrás | `git branch -vv` |
| Traer cambios y limpiar ramas remotas borradas | `git fetch --prune` |
| Actualizar sin commits de merge innecesarios | `git pull --rebase` |
| Volver a la rama anterior | `git switch -` |
| Guardar cambios a medias | `git stash push -m "wip login"` / `git stash pop` |
| Traer un commit concreto a otra rama | `git cherry-pick -x <sha>` |
| Poner mi feature al día con develop (rama solo mía) | `git rebase develop` |
| Subir tras un rebase | `git push --force-with-lease` (nunca `--force` a secas) |
| Ver qué cambió entre dos versiones | `git diff v1.1.0..v1.2.0 --stat` |
| ¿Quién cambió esta línea? | `git blame -L 40,60 fichero` |

### Deshacer

| Situación | Comando |
|---|---|
| Descartar cambios de un fichero sin commitear | `git restore fichero` |
| Sacar un fichero del stage | `git restore --staged fichero` |
| Cambiar el mensaje del último commit (sin push) | `git commit --amend` |
| Deshacer el último commit manteniendo los cambios (sin push) | `git reset --soft HEAD~1` |
| Deshacer un commit **ya publicado** | `git revert <sha>` (crea un commit inverso) |
| Revertir un merge publicado | `git revert -m 1 <sha-del-merge>` |
| "He perdido commits" tras un reset o rebase | `git reflog` → `git switch -c rescate <sha>` |

> Regla de oro: **nunca reescribir historia publicada** en ramas compartidas (`main`,
> `develop`). Ahí se usa `revert`; `reset` y `rebase`, solo en tus ramas locales.

## Conflictos

```bash
git merge feature/login           # CONFLICT (content): Merge conflict in app.yml
git status                        # ficheros en "both modified"
```

```
<<<<<<< HEAD
puerto: 8080
=======
puerto: 9090
>>>>>>> feature/login
```

1. Editar el fichero dejando la versión correcta (sin los marcadores).
2. `git add app.yml`
3. `git commit` (o `git rebase --continue` si estabas en un rebase).
4. Si te lías: `git merge --abort` / `git rebase --abort` y vuelta al estado anterior.
