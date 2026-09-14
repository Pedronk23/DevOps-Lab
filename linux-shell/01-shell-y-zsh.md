# Shell — zsh y personalización

## Terminal, consola y shell no son lo mismo

```
   ┌──────────── TERMINAL (emulador) ────────────┐
   │  Windows Terminal, GNOME Terminal, iTerm2   │  ← la ventana: dibuja texto,
   │                                             │    fuentes, colores, pestañas
   │   ┌──────────── SHELL ──────────────────┐   │
   │   │  bash, zsh, fish, sh, PowerShell    │   │  ← el intérprete: lee lo que
   │   │                                     │   │    escribes y lanza programas
   │   │   ls, kubectl, ansible…  (programas)│   │
   │   └─────────────────────────────────────┘   │
   └─────────────────────────────────────────────┘
```

- **Terminal**: la aplicación gráfica. Cambiar la fuente o el tema de colores se hace aquí.
- **Shell**: el programa que interpreta comandos. Los alias, el prompt y el autocompletado se
  configuran aquí.
- **Consola**: históricamente, la terminal física del sistema (las TTY de `Ctrl+Alt+F3`).

## Cambiar de shell

```bash
chsh -s /bin/zsh
```

Ver la shell activa:

```bash
echo $SHELL
```

| Comando | Qué muestra |
|---|---|
| `echo $SHELL` | tu shell **de login** (la configurada en `/etc/passwd`) |
| `echo $0` | la shell que **estás usando ahora** (si lanzaste `bash` desde zsh, dice `bash`) |
| `cat /etc/shells` | shells válidas para `chsh` |
| `ps -p $$` | el proceso de la shell actual |

> `chsh` hace efecto en el **siguiente login**: cierra la sesión (o abre una SSH nueva).

## Ficheros de arranque

Cada shell lee unos ficheros según **cómo** se abre:

| Tipo de sesión | Ejemplo |
|---|---|
| **login** | entrar por SSH, TTY, `su -` |
| **interactiva no-login** | abrir una pestaña nueva de la terminal en el escritorio |
| **no interactiva** | ejecutar un script |

| Shell | Fichero | Cuándo se lee |
|---|---|---|
| bash | `/etc/profile`, `~/.bash_profile` o `~/.profile` | login |
| bash | `~/.bashrc` | interactiva no-login (muchas distros lo cargan también desde `.bash_profile`) |
| zsh | `~/.zshenv` | **siempre**, incluidos scripts (solo variables, nada que imprima) |
| zsh | `~/.zprofile` | login |
| zsh | `~/.zshrc` | interactiva: alias, prompt, plugins |

```bash
source ~/.zshrc      # recargar la configuración sin abrir otra terminal
exec zsh            # reemplazar la shell actual por una limpia
```

## Personalización

- **Tipografía**: se cambia desde los ajustes de la terminal (recomendada: *MesloLGS NF*,
  necesaria para que los temas con iconos se vean bien).
- Configuración del usuario:

```bash
nano .zshrc
```

Dentro del `.zshrc`:

- `THEME`: por ejemplo `agnoster`.
- **Oh My Zsh**: framework con plugins y temas.
- **Plugins** habituales:
  - `zsh-autosuggestions` → sugerencias según el histórico.
  - `zsh-syntax-highlighting` → resaltado de sintaxis.
- `git` / git-flow: plugin útil para trabajar con ramas.

### Oh My Zsh

```bash
# instalar (requiere zsh, git y curl)
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# plugins externos: se clonan en la carpeta custom
git clone https://github.com/zsh-users/zsh-autosuggestions \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

Un `.zshrc` típico:

```bash
export ZSH="$HOME/.oh-my-zsh"
ZSH_THEME="agnoster"                       # o "robbyrussell", "powerlevel10k/powerlevel10k"

plugins=(git docker kubectl)               # plugins incluidos en Oh My Zsh

source $ZSH/oh-my-zsh.sh

# plugins externos (syntax-highlighting debe ir el ÚLTIMO)
source ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh
source ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
```

| Alternativa | Qué es |
|---|---|
| **Powerlevel10k** | tema muy rápido y configurable (asistente: `p10k configure`) |
| **Starship** | prompt multiplataforma (bash, zsh, fish, PowerShell) con un único `starship.toml` |
| **zinit / sheldon** | gestores de plugins más ligeros que Oh My Zsh |

> Si la terminal tarda en abrir, el culpable suele ser un plugin. Mide con
> `time zsh -i -c exit`.

## Configuración útil para DevOps

```bash
# PATH: añadir binarios propios delante
export PATH="$HOME/.local/bin:$PATH"
export EDITOR=vim

# alias
alias k=kubectl
alias kgp='kubectl get pods'
alias tf=terraform
alias ll='ls -lah'

# autocompletado de kubectl (y que funcione con el alias k)
source <(kubectl completion zsh)
compdef k=kubectl

# historial grande y compartido entre pestañas
HISTSIZE=50000
SAVEHIST=50000
setopt SHARE_HISTORY HIST_IGNORE_DUPS HIST_IGNORE_SPACE
```

> Con `HIST_IGNORE_SPACE`, un comando que empieza por **espacio** no se guarda en el
> historial: útil cuando tienes que escribir un token a mano. En bash el equivalente es
> `HISTCONTROL=ignorespace`.

## Comparativa de shells

| Shell | Notas |
|---|---|
| `bash` y `zsh` | **comparten scripts**: un script de bash funciona normalmente en zsh |
| `fish` | más moderno y amigable, pero **no comparte** la sintaxis de scripts |

### Matiz: "normalmente" no es "siempre"

La mayoría de scripts sencillos funcionan igual, pero hay diferencias que muerden:

| Situación | bash | zsh |
|---|---|---|
| Índice de arrays | empieza en `0` | empieza en `1` |
| Variable sin comillas con espacios | se parte en palabras | **no** se parte |
| Comodín que no encuentra nada (`ls *.log`) | pasa el texto literal | error `no matches found` |
| `?` y `[` en argumentos sin comillas | normal | se interpretan como comodines |

```bash
# en zsh esto falla con "no matches found" por el "?":
curl https://api.lab.local/items?page=2
# solución: comillas
curl "https://api.lab.local/items?page=2"
```

### El shebang manda

Lo que decide con qué shell se ejecuta un script es su **primera línea**, no la shell que
tengas abierta:

```bash
#!/usr/bin/env bash
set -euo pipefail     # para al primer error, variable sin definir = error, fallos en pipes
```

| Shebang | Notas |
|---|---|
| `#!/usr/bin/env bash` | busca bash en el PATH: el más portable |
| `#!/bin/bash` | ruta fija |
| `#!/bin/sh` | shell POSIX mínima; en Debian/Ubuntu es **dash**, no bash |

> Error clásico: un script con sintaxis de bash (`[[ ]]`, arrays, `source`) lanzado con
> `sh script.sh`. En Debian falla porque `sh` es dash. Lánzalo con `bash script.sh` o
> `./script.sh`.

Para revisar scripts: `shellcheck script.sh` detecta comillas olvidadas, *bashisms* y errores
típicos antes de ejecutarlos.
