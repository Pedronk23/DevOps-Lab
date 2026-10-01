# Linux — nano

**nano** es el editor de texto en terminal más sencillo: abres el fichero y escribes, sin modos.
Los atajos principales aparecen siempre en las dos líneas inferiores de la pantalla, así que no
hace falta memorizar nada para empezar. Viene instalado en Debian, Ubuntu y derivadas (y suele
ser su editor por defecto); en instalaciones mínimas de RHEL/Rocky a veces no está:
`sudo dnf install nano`.

Es la opción cómoda para cambios puntuales en un fichero de configuración. Para editar mucho y a
menudo por SSH compensa aprender [vim](06-vim.md).

## Cómo leer los atajos

```
  GNU nano 7.2                      /etc/hosts                        Modified

127.0.0.1   localhost
10.0.0.15   db01.lab.local db01


^G Help        ^O Write Out   ^W Where Is    ^K Cut         ^T Execute     ^C Location
^X Exit        ^R Read File   ^\ Replace     ^U Paste       ^J Justify     ^/ Go To Line
```

| Símbolo | Tecla |
|---|---|
| `^` | `Ctrl` (`^X` = `Ctrl + X`) |
| `M-` | `Alt` (`M-U` = `Alt + U`). Si `Alt` no funciona (algunas terminales, macOS), pulsa y suelta `Esc` y después la tecla |

Con el sistema en español los textos salen traducidos (`^O Guardar`, `^X Salir`,
`^W Buscar`…), pero los atajos son los mismos. `Modified` arriba a la derecha indica que hay
cambios sin guardar.

## Abrir ficheros

```bash
nano fichero.conf          # abrir (o crear si no existe)
nano +42 fichero.conf      # abrir en la línea 42
nano -l fichero.conf       # con números de línea
nano -v fichero.log        # solo lectura
nano -B fichero.conf       # al guardar, deja una copia del original (fichero.conf~)
sudoedit /etc/hosts        # fichero de root, con tu configuración (o sudo nano /etc/hosts)
```

> Las versiones anteriores a nano 4.0 (p. ej. la de CentOS 7) **parten las líneas largas** al
> escribir, y eso puede romper un fichero de configuración. Ábrelos con `nano -w` para
> evitarlo. Desde la 4.0 ya no lo hace por defecto.

## Guardar y salir

| Atajo | Acción |
|---|---|
| `Ctrl + O`, luego `Enter` | guardar (*Write Out*); `Enter` confirma el nombre del fichero |
| `Ctrl + S` | guardar sin preguntar |
| `Ctrl + X` | salir. Si hay cambios, pregunta si guardarlos: `Y` sí (`S` con el sistema en español), `N` no, `Ctrl + C` cancelar |

> Para guardar con otro nombre: `Ctrl + O`, cambia el nombre en la línea inferior y `Enter`.

## Moverse

| Atajo | Movimiento |
|---|---|
| flechas | carácter a carácter y línea a línea |
| `Ctrl + ←` / `Ctrl + →` | palabra anterior / siguiente |
| `Ctrl + A` / `Ctrl + E` | principio / final de la línea (igual que en la shell) |
| `Ctrl + Y` / `Ctrl + V` | página arriba / abajo (también `RePág` / `AvPág`) |
| `Alt + \` / `Alt + /` | principio / final del fichero (también `Ctrl + Inicio` / `Ctrl + Fin`) |
| `Ctrl + _` (o `Ctrl + /`) | ir a una línea; admite línea y columna: `42,5` |
| `Alt + ]` | saltar al paréntesis, corchete o llave pareja |
| `Ctrl + C` | mostrar en qué línea y columna estás |

## Editar: cortar, copiar y pegar

nano no tiene un portapapeles por selección como un editor gráfico: se trabaja **por líneas**
o marcando un bloque.

| Atajo | Acción |
|---|---|
| `Ctrl + K` | cortar la línea actual (o el texto marcado) |
| `Alt + 6` | copiar la línea actual (o el texto marcado) |
| `Ctrl + U` | pegar lo último cortado o copiado |
| `Alt + A` (o `Ctrl + 6`) | empezar a marcar texto; después se amplía la selección con las flechas |
| `Alt + T` | cortar desde el cursor hasta el final del fichero |
| `Alt + U` / `Alt + E` | deshacer / rehacer |
| `Alt + }` / `Alt + {` | indentar / desindentar la línea o el bloque marcado |
| `Alt + 3` | comentar / descomentar la línea o el bloque marcado |

**Mover un bloque de líneas**: pulsa `Ctrl + K` varias veces seguidas (las líneas cortadas se
acumulan), ve al destino y pega todas juntas con `Ctrl + U`.

## Buscar y reemplazar

| Atajo | Acción |
|---|---|
| `Ctrl + W` | buscar (*Where Is*) |
| `Alt + W` / `Alt + Q` | siguiente / anterior resultado |
| `Ctrl + \` | reemplazar: pide el texto a buscar, el nuevo, y pregunta en cada coincidencia (sí, no o todas) |

Opciones dentro de la búsqueda o el reemplazo (antes de pulsar `Enter`):

| Atajo | Activa |
|---|---|
| `Alt + C` | distinguir mayúsculas y minúsculas |
| `Alt + R` | expresiones regulares (ver [regex](../fundamentos/regex-basico.md)) |
| `Alt + B` | buscar hacia atrás |

## Otras utilidades

| Atajo | Acción |
|---|---|
| `Ctrl + G` | ayuda con todos los atajos |
| `Ctrl + R` | insertar otro fichero en la posición del cursor |
| `Ctrl + T` | ejecutar un comando e insertar su salida (nano 5 o posterior; antes era el corrector ortográfico) |
| `Alt + N` | mostrar u ocultar los números de línea (`Alt + #` en versiones antiguas) |
| `Alt + P` | mostrar tabuladores y espacios |
| `Ctrl + J` | justificar el párrafo: **une líneas**. Si lo pulsas sin querer en un fichero de configuración, `Alt + U` lo deshace |

## Configuración: `~/.nanorc`

```
# números de línea siempre
set linenumbers
# mostrar siempre la línea y columna del cursor
set constantshow
# Tab de 2 espacios, insertando espacios (imprescindible en YAML)
set tabsize 2
set tabstospaces
# la línea nueva mantiene la indentación de la anterior
set autoindent
# las líneas largas se ven partidas en pantalla, pero NO se parten en el fichero
set softwrap
# resaltado de sintaxis (YAML, sh, Dockerfile, nginx…)
include "/usr/share/nano/*.nanorc"
```

En muchas distribuciones el resaltado de sintaxis ya viene activado desde `/etc/nanorc`.

> Desde nano 8, `nano --modernbindings` (o `set modernbindings` en el `.nanorc`) cambia a los
> atajos de los editores gráficos: `Ctrl + Q` salir, `Ctrl + F` buscar, `Ctrl + C` / `Ctrl + V`
> copiar y pegar, `Ctrl + Z` deshacer.

## Elegir el editor por defecto

Muchas herramientas abren un editor por su cuenta: `git commit`, `crontab -e`, `visudo`,
`kubectl edit`, `systemctl edit`, `sudoedit`. Cuál se abre depende de estas variables y
ajustes:

```bash
# en ~/.bashrc o ~/.zshrc
export EDITOR=nano            # el que usan casi todas las herramientas
export VISUAL=nano            # algunas la miran antes que EDITOR
export KUBE_EDITOR=nano       # solo para kubectl edit
git config --global core.editor nano

# Debian/Ubuntu
sudo update-alternatives --config editor   # editor por defecto del sistema
select-editor                              # elección por usuario (lo que pregunta crontab -e la primera vez)
```

## ¿nano o vim?

| | nano | [vim](06-vim.md) |
|---|---|---|
| Aprendizaje | ninguno: abres y escribes | alto: hay que entender los modos |
| Disponibilidad | casi siempre en Debian/Ubuntu; no siempre en sistemas mínimos | `vi` está en prácticamente cualquier Unix (en Alpine, el de BusyBox) |
| Velocidad editando | normal | muy alta una vez aprendido |
| Buscar y reemplazar | básico, con regex | muy potente (`:%s`, `:g`, macros) |
| Ideal para | cambios puntuales en un fichero de configuración | editar mucho y a menudo por SSH |

> En contenedores mínimos (Alpine, *distroless*) a menudo no hay ni uno ni otro. Mejor no
> editar dentro del contenedor: cambia la imagen o el ConfigMap.

## Chuleta

```
GUARDAR/SALIR   ^O guardar · ^S guardar ya · ^X salir
MOVERSE         ^A ^E línea · ^Y ^V página · M-\ M-/ fichero · ^_ ir a línea · M-] pareja
EDITAR          ^K cortar · M-6 copiar · ^U pegar · M-A marcar · M-U M-E deshacer/rehacer
BUSCAR          ^W buscar · M-W M-Q siguiente/anterior · ^\ reemplazar
OTROS           ^G ayuda · ^C posición · M-3 comentar · M-N números de línea
```
