# Claude Code (CLI) — notas de uso

Notas propias de configuración y uso del CLI en un entorno Debian.

> Es una herramienta que cambia rápido: comandos y atajos pueden variar entre versiones.
> Ante la duda, `/help` dentro de la sesión manda.

## Instalación y mantenimiento

```bash
curl -fsSL https://claude.ai/install.sh | bash      # instalador nativo (Linux/macOS)
claude --version
claude update                                        # actualizar
claude doctor                                        # diagnosticar la instalación
```

(En Windows existe instalador para PowerShell y también funciona dentro de WSL.)

## Básico

| Acción | Cómo |
|---|---|
| Entrar | `claude` |
| Salir | `exit` |
| Comandos internos | con `/` → `/skills`, `/agents`, `/clear`, … |
| Nivel de razonamiento | `/effort` (lo dejo en *medium* por defecto) |

### Formas de arrancar

| Comando | Qué hace |
|---|---|
| `claude` | sesión interactiva en la carpeta actual |
| `claude "explica la estructura de este repo"` | arranca con una primera petición |
| `claude -c` | continúa la última conversación de esta carpeta |
| `claude -r` | elige una conversación anterior para retomarla |
| `claude -p "pregunta"` | modo no interactivo: responde y sale (scripts, CI) |
| `cat error.log \| claude -p "¿qué falla?"` | pasarle datos por la tubería |

> Arranca `claude` **en la raíz del repositorio**: es su punto de partida para leer código y
> donde busca el `CLAUDE.md` del proyecto.

## Comandos de sesión útiles

| Comando | Para qué |
|---|---|
| `/help` | lista de comandos disponibles en tu versión |
| `/clear` | conversación nueva (hazlo al cambiar de tarea: menos ruido, mejores respuestas) |
| `/compact` | resume la conversación para liberar contexto sin perder el hilo |
| `/context` | cuánto contexto se está usando |
| `/model` | cambiar de modelo |
| `/config` | ajustes |
| `/permissions` | qué herramientas y comandos puede usar sin preguntar |
| `/memory` | editar los ficheros de memoria (`CLAUDE.md`) |
| `/init` | genera un `CLAUDE.md` analizando el repositorio |
| `/resume` | retomar una conversación anterior |
| `/rewind` | volver a un punto anterior de la conversación (y del código) |
| `/mcp` | servidores MCP conectados |
| `/agents` / `/skills` | subagentes y skills disponibles |

## Atajos de teclado

| Atajo | Acción |
|---|---|
| `Esc` | interrumpir lo que está haciendo (sin perder la conversación) |
| `Esc` `Esc` | volver atrás a un mensaje anterior |
| `Shift + Tab` | alternar modos: normal → aceptar ediciones automáticamente → **plan** |
| `@ruta/fichero` | referenciar un fichero en el mensaje (autocompleta con `Tab`) |
| `!comando` | ejecutar un comando de shell directamente y que vea la salida |
| `\` + `Enter` | salto de línea sin enviar (según terminal, también `Shift + Enter`; `/terminal-setup`) |
| `Ctrl + C` | cancelar la entrada; dos veces, salir |

### Modo plan

Con `Shift + Tab` hasta **plan mode**: investiga y propone un plan **sin tocar nada**. Para
cambios grandes (refactorizar un rol, migrar manifiestos) es la forma segura de empezar: lees
el plan, lo corriges y después le dejas ejecutar.

## Texto sin formato

Para escribir un bloque de texto plano (sin que se interprete el formato), se abre con
**tres comillas invertidas** antes de enviar:

````
```
texto plano, formato, etc.
```
````

Útil para pegar logs, trazas o YAML sin que se "coman" los espacios o los caracteres
especiales.

## Memoria: `CLAUDE.md`

Ficheros Markdown con instrucciones que se cargan **al empezar cada sesión**.

| Fichero | Alcance | ¿A Git? |
|---|---|---|
| `~/.claude/CLAUDE.md` | todos tus proyectos (preferencias personales) | no |
| `./CLAUDE.md` | este repositorio (convenciones del equipo) | **sí** |
| `./subcarpeta/CLAUDE.md` | se añade al trabajar en esa carpeta | sí |

Qué poner en el `CLAUDE.md` de un repositorio de infraestructura:

```markdown
# Proyecto: roles de Ansible del equipo

## Comandos
- Lint: `ansible-lint`
- Tests de un rol: `cd roles/<rol> && molecule test`

## Convenciones
- Módulos siempre con FQCN (`ansible.builtin.copy`, nunca `copy`).
- Variables del rol con prefijo del rol: `nginx_puerto`.
- Nada de `shell`/`command` si existe módulo; si no, `changed_when` obligatorio.

## Prohibido
- Tocar `inventories/produccion/`.
- Escribir secretos en claro: van en Ansible Vault.
```

> Corto y concreto funciona mejor que largo: es contexto que se consume en cada sesión.

## Permisos y ficheros de configuración

| Fichero | Alcance |
|---|---|
| `~/.claude/settings.json` | usuario, todos los proyectos |
| `.claude/settings.json` | proyecto, compartido con el equipo (a Git) |
| `.claude/settings.local.json` | proyecto, solo tú (no se sube) |

```json
{
  "permissions": {
    "allow": [
      "Bash(git status)",
      "Bash(git diff:*)",
      "Bash(ansible-lint:*)",
      "Bash(kubectl get:*)"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./**/*secret*)",
      "Bash(kubectl delete:*)",
      "Bash(terraform apply:*)"
    ]
  }
}
```

- `allow`: no pregunta antes de ejecutarlos.
- `deny`: no puede usarlos aunque se lo pidas.
- Lo que no está en ninguna lista, **lo pregunta**.

> Regla personal: todo lo de solo lectura (`get`, `describe`, `diff`, `lint`) en `allow`;
> todo lo que cambia un clúster o un entorno (`apply`, `delete`, `helm upgrade`,
> `ansible-playbook` contra producción), **nunca** en `allow`.

## Skills

- Se pueden instalar skillsets de terceros publicados en repositorios.
- En la versión web también se pueden añadir skills propias, por ejemplo:
  - `/caveman`
  - `/grill-me`

Una skill es una carpeta con un `SKILL.md`: instrucciones que se cargan cuando la tarea
encaja con su descripción, o al invocarla con `/nombre`.

```
.claude/skills/revisar-rol/SKILL.md       ← del proyecto
~/.claude/skills/revisar-rol/SKILL.md     ← personal, en todos los proyectos
```

```markdown
---
name: revisar-rol
description: Revisa un rol de Ansible contra las convenciones del equipo. Úsala cuando se pida revisar o auditar un rol.
---

1. Ejecuta `ansible-lint roles/<rol>`.
2. Comprueba FQCN, prefijos de variables y `changed_when` en command/shell.
3. Resume los problemas en una tabla: fichero, línea, problema, propuesta.
```

Los skillsets de terceros se distribuyen como **plugins** desde un *marketplace* (un
repositorio): se añaden y se instalan con `/plugin`.

## Subagentes y MCP

| Pieza | Qué es | Dónde |
|---|---|---|
| **Subagente** | un asistente especializado con sus propias instrucciones y herramientas, al que se delega una subtarea | `.claude/agents/*.md`, gestionados con `/agents` |
| **MCP** (*Model Context Protocol*) | conectores a sistemas externos: GitLab, bases de datos, navegador, documentación | `claude mcp add …`, `.mcp.json` en el proyecto |
| **Hooks** | comandos que se ejecutan automáticamente en ciertos eventos (p. ej. formatear tras cada edición) | `settings.json` |

```bash
claude mcp list
claude mcp add <nombre> -- <comando que arranca el servidor MCP>
```

> Un servidor MCP tiene los permisos de las credenciales que le des. Uno de GitLab con un
> token de administrador puede hacer cualquier cosa que ese token permita.

## Notas de configuración

- Integración con el repositorio de Git del equipo (GitLab) para trabajar sobre las ramas.

Flujo que uso con GitLab:

```
   1. git switch -c feature/PROJ-123-...        (o se lo pido a Claude)
   2. le explico la tarea, en plan mode si es grande
   3. reviso el diff:  git diff                  ← siempre, antes de commitear
   4. commit (convencional) y push
   5. MR con glab:     glab mr create --fill --target-branch develop
   6. la revisión humana del MR sigue siendo obligatoria
```

Con `glab` (CLI de GitLab) instalado y autenticado, Claude puede crear la MR, leer los
comentarios de revisión o ver por qué falló una pipeline (`glab ci view`). Ver
[Git Flow](../git/gitflow.md).

## Buenas prácticas

- **Commit antes de pedir un cambio grande**: si no te gusta el resultado, `git restore .` y
  vuelta a empezar.
- Una tarea por conversación; `/clear` al cambiar de tema.
- Dale la forma de **verificar** su trabajo: el comando de lint, de tests o de Molecule. Rinde
  mucho mejor cuando puede comprobar si lo que hizo funciona.
- Pídele que **lea antes de cambiar** ("mira cómo están hechos los otros roles y sigue el
  mismo patrón").
- Revisar todo lo que toca infraestructura real como revisarías el MR de un compañero.
- No pegar secretos en la conversación; si hace falta un token, que lo lea de una variable
  de entorno.
