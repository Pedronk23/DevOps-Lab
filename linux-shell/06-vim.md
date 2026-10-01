# Linux — vim

**vim** (*Vi IMproved*) es el editor de texto en terminal que vas a encontrar en prácticamente
cualquier servidor Linux: si no está vim, al menos está `vi`. Funciona por **modos**: al abrir
un fichero las teclas no escriben texto, sino que son **órdenes** (moverse, borrar, copiar…)
hasta que entras en modo inserción. Cuesta al principio, pero se edita muy rápido sin soltar el
teclado, y por SSH a menudo es lo único que hay.

> Para aprender desde cero: `vimtutor`, un tutorial interactivo de unos 30 minutos que viene
> con vim (`vimtutor es` para la versión en español, si está instalada).

## Lo mínimo para sobrevivir

```
vim fichero  →  i  (escribir)  →  Esc  →  :wq  (guardar y salir)
                                          :q!  (salir SIN guardar)
```

> Si no sabes en qué modo estás o las teclas "hacen cosas raras", pulsa `Esc` un par de veces:
> vuelves al modo normal.

## Modos

```
                 i a o                      :
 INSERCIÓN  ◄──────────────  NORMAL  ──────────────►  LÍNEA DE ÓRDENES
 (escribir) ───── Esc ─────►   │ ▲   ◄─ Enter/Esc ──  (:w  :q  :%s)
                    v V Ctrl+v │ │ Esc
                               ▼ │
                             VISUAL (seleccionar)
```

| Modo | Para qué | Cómo se entra | Cómo se sale |
|---|---|---|---|
| **Normal** | moverse, borrar, copiar, pegar | es el modo al abrir | — |
| **Inserción** | escribir texto | `i`, `a`, `o`… | `Esc` |
| **Visual** | seleccionar texto | `v` (caracteres), `V` (líneas), `Ctrl + V` (bloque) | `Esc` |
| **Línea de órdenes** | guardar, salir, buscar y reemplazar | `:` | `Enter` (ejecuta) o `Esc` (cancela) |

En la parte inferior de la pantalla se ve el modo activo: `-- INSERT --`, `-- VISUAL --`
(`-- INSERTAR --` con el sistema en español).

## Abrir ficheros

```bash
vim fichero.yaml               # abrir (o crear si no existe)
vim +42 fichero.yaml           # abrir en la línea 42 (p. ej. la que marca un error)
vim +/replicas deploy.yaml     # abrir en la primera aparición de "replicas"
vim -R fichero.log             # solo lectura (equivale a `view`)
vim -O a.yaml b.yaml           # dos ficheros lado a lado
vimdiff a.conf b.conf          # comparar dos ficheros (equivale a `vim -d`)
sudoedit /etc/ssh/sshd_config  # editar un fichero de root con TU configuración de vim
```

## Guardar y salir

| Orden | Acción |
|---|---|
| `:w` | guardar |
| `:q` | salir (solo si no hay cambios sin guardar) |
| `:wq` / `:x` / `ZZ` | guardar y salir (`:x` y `ZZ` solo escriben si hay cambios) |
| `:q!` / `ZQ` | salir **descartando** los cambios |
| `:w nuevo.yaml` | guardar con otro nombre |
| `:e!` | descartar los cambios y recargar el fichero desde disco |
| `:wa` / `:qa` / `:wqa` | lo mismo, con todos los ficheros abiertos |

> ¿Abriste un fichero de root sin `sudo` y no te deja guardar (`E45: 'readonly' option is set`
> o `E212: Can't open file for writing`)? `:w !sudo tee % > /dev/null` lo guarda con sudo sin
> perder los cambios. Vim avisará de que el fichero ha cambiado en disco: pulsa `L` para
> recargarlo y sal con `:q`. (Funciona en vim, no en Neovim.)

## Entrar en modo inserción

| Tecla | Empieza a escribir… |
|---|---|
| `i` | antes del cursor |
| `a` | después del cursor |
| `I` | al principio de la línea |
| `A` | al final de la línea |
| `o` | en una línea nueva **debajo** |
| `O` | en una línea nueva **encima** |
| `s` | sustituyendo el carácter bajo el cursor |
| `S` | sustituyendo la línea entera |

## Moverse (modo normal)

```
        k
        ▲
   h ◄     ► l
        ▼
        j
```

| Tecla | Movimiento |
|---|---|
| `h` `j` `k` `l` | izquierda, abajo, arriba, derecha (las flechas también funcionan) |
| `w` / `b` / `e` | siguiente palabra / palabra anterior / final de la palabra |
| `W` / `B` | igual, pero "palabra" es todo hasta el siguiente espacio (útil con rutas y URLs) |
| `0` / `^` / `$` | principio de la línea / primer carácter que no es espacio / final de la línea |
| `gg` / `G` | principio / final del fichero |
| `42G` o `:42` | ir a la línea 42 |
| `{` / `}` | bloque anterior / siguiente (salta entre líneas en blanco) |
| `Ctrl + D` / `Ctrl + U` | media página abajo / arriba |
| `Ctrl + F` / `Ctrl + B` | página entera abajo / arriba |
| `H` / `M` / `L` | arriba / centro / abajo de la pantalla |
| `zz` | centrar la pantalla en el cursor |
| `%` | saltar al paréntesis, corchete o llave pareja |
| `f<c>` / `t<c>` | saltar a / justo antes de la siguiente aparición del carácter `<c>` en la línea (`;` repite) |
| `Ctrl + O` / `Ctrl + I` | volver a la posición anterior / siguiente (después de un salto o una búsqueda) |

> Casi todo admite un **número delante**: `5j` baja 5 líneas, `3w` avanza 3 palabras.

## Editar (modo normal)

| Tecla | Acción |
|---|---|
| `x` | borrar el carácter bajo el cursor |
| `r<c>` | reemplazar el carácter bajo el cursor por `<c>` |
| `dd` | cortar la línea |
| `yy` | copiar la línea |
| `p` / `P` | pegar después / antes del cursor (debajo / encima si es una línea entera) |
| `D` / `C` | borrar / cambiar desde el cursor hasta el final de la línea |
| `J` | unir la línea siguiente a la actual |
| `>>` / `<<` | indentar / desindentar la línea |
| `~` | cambiar entre mayúsculas y minúsculas |
| `u` | deshacer |
| `Ctrl + R` | rehacer |
| `.` | repetir el último cambio |

En vim, "borrar" es en realidad **cortar**: lo que se borra con `d` o `x` se puede pegar con `p`.
Por eso `ddp` intercambia una línea con la siguiente.

### La gramática: operador + movimiento

Las órdenes se combinan como frases: **[número] + operador + movimiento u objeto**.

| Operador | Hace | | Movimiento u objeto | Sobre |
|---|---|---|---|---|
| `d` | borrar (cortar) | | `w` | hasta la siguiente palabra |
| `c` | cambiar (borra y entra en inserción) | | `$` | hasta el final de la línea |
| `y` | copiar | | `iw` / `aw` | la palabra (sin / con el espacio de después) |
| `>` / `<` | indentar / desindentar | | `i"` / `a"` | el texto entre comillas (sin / con las comillas) |
| | | | `i(` `i{` `i[` | el contenido de paréntesis, llaves o corchetes |
| | | | `ip` | el bloque actual (hasta la línea en blanco) |
| | | | `G` / `gg` | hasta el final / principio del fichero |

| Orden | Qué hace |
|---|---|
| `3dd` | corta 3 líneas |
| `d3j` | corta la línea actual y las 3 siguientes |
| `dw` | borra hasta la siguiente palabra |
| `ciw` | cambia la palabra bajo el cursor (*change inner word*) |
| `ci"` | cambia el texto entre comillas: perfecto para `image: "nginx:1.25"` |
| `ct:` | cambia hasta los dos puntos (*change till*) |
| `yip` | copia el bloque actual |
| `>ip` | indenta el bloque entero |
| `dG` | borra desde la línea actual hasta el final del fichero |

## Buscar y reemplazar

| Orden | Acción |
|---|---|
| `/texto` | buscar hacia delante |
| `?texto` | buscar hacia atrás |
| `n` / `N` | siguiente / anterior resultado |
| `*` / `#` | buscar la palabra bajo el cursor, hacia delante / atrás |
| `/texto\c` | buscar sin distinguir mayúsculas y minúsculas |
| `:noh` | quitar el resaltado de la última búsqueda |
| `:s/viejo/nuevo/` | reemplazar la primera aparición en la línea actual |
| `:s/viejo/nuevo/g` | todas las de la línea actual |
| `:%s/viejo/nuevo/g` | todas las del fichero |
| `:%s/viejo/nuevo/gc` | todas, **preguntando** una a una (`y` sí, `n` no, `a` todas, `q` salir) |
| `:10,20s/viejo/nuevo/g` | solo entre las líneas 10 y 20 |
| `:g/patrón/d` | borrar todas las líneas que contienen el patrón |
| `:v/patrón/d` | borrar todas las líneas que **no** lo contienen |

> La sintaxis de `:s` es la misma que la de [sed](05-sed.md): `s/buscar/reemplazar/flags`. Con
> una selección visual hecha, al pulsar `:` aparece `:'<,'>` y la orden se aplica solo a esas
> líneas.

## Modo visual

| Tecla | Selecciona |
|---|---|
| `v` | por caracteres |
| `V` | por líneas enteras |
| `Ctrl + V` | un bloque rectangular (por columnas) |
| `gv` | vuelve a seleccionar lo último que seleccionaste |

Con la selección hecha: `d` corta, `y` copia, `c` cambia, `>` / `<` indenta y `:` aplica una
orden solo a esas líneas.

**Comentar varias líneas a la vez** (muy útil en ficheros de configuración):

```
Ctrl + V    → modo bloque
j j j       → baja seleccionando la primera columna de las líneas
I           → insertar al principio del bloque
#           → escribe el carácter de comentario
Esc         → se aplica a todas las líneas seleccionadas
```

Para descomentar: `Ctrl + V`, seleccionar la columna de `#` y pulsar `d`.

## Varios ficheros y comandos de shell

| Orden | Acción |
|---|---|
| `:e otro.yaml` | abrir otro fichero |
| `:sp fichero` / `:vsp fichero` | dividir la pantalla en horizontal / vertical |
| `Ctrl + W`, luego `w` | pasar a la siguiente ventana (o `Ctrl + W` + `h`/`j`/`k`/`l`) |
| `:r fichero` | insertar el contenido de otro fichero debajo del cursor |
| `:r !comando` | insertar la salida de un comando (`:r !date`, `:r !hostname -I`) |
| `:!comando` | ejecutar un comando sin salir de vim (`:!kubectl apply -f %`) |

En la línea de órdenes, `%` es el fichero actual.

## Configuración: `~/.vimrc`

Los ajustes se pueden activar al vuelo con `:set …` o dejarse fijos en `~/.vimrc`:

```vim
" resaltado de sintaxis
syntax on
set number                 " números de línea
set relativenumber         " números relativos: ayudan con 5j, 3dd…
set hlsearch incsearch     " resaltar resultados y buscar mientras escribes
set ignorecase smartcase   " sin distinguir mayúsculas, salvo si escribes alguna
set expandtab              " Tab inserta espacios (imprescindible en YAML)
set tabstop=2 shiftwidth=2
set autoindent
set list listchars=tab:»·,trail:·   " ver tabuladores y espacios al final de línea

autocmd FileType yaml setlocal tabstop=2 softtabstop=2 shiftwidth=2 expandtab
```

| Ajuste al vuelo | Para qué |
|---|---|
| `:set paste` / `:set nopaste` | pegar sin que vim reindente el texto "en escalera" (pasa mucho al pegar por SSH) |
| `:set ff?` / `:set ff=unix` | ver / convertir los finales de línea de Windows (CRLF) a Linux (LF); luego `:w` |
| `:set nobomb` | quitar el BOM de un fichero UTF-8; luego `:w` (ver [parsers y codificación](../fundamentos/parsers-y-codificacion.md)) |
| `:set list` | mostrar tabuladores (`^I`) y finales de línea (`$`) |
| `:help tema` | ayuda integrada (`:help :s`, `:help text-objects`) |

## Avisos que asustan

| Mensaje o síntoma | Qué pasa | Qué hacer |
|---|---|---|
| `E325: ATTENTION` … `Found a swap file` | existe `.fichero.swp`: otra sesión está editando el fichero, o vim se cerró mal (p. ej. se cortó el SSH) | si nadie más lo edita: `R` para recuperar, revisar y guardar, y después borrar el `.swp`. `D` lo borra si no hay nada que recuperar. `O` abre en solo lectura |
| `E37: No write since last change` | intentas salir con cambios sin guardar | `:wq` para guardar o `:q!` para descartarlos |
| `E45: 'readonly' option is set` | fichero de solo lectura o de root | el truco de `sudo tee` de [Guardar y salir](#guardar-y-salir) |
| la pantalla no responde | pulsaste `Ctrl + S`, que congela la terminal | `Ctrl + Q` (ver [atajos de consola](02-atajos-consola.md)) |
| vim "desaparece" | pulsaste `Ctrl + Z` y quedó suspendido | `fg` para volver |
| abajo pone `recording @q` | pulsaste `q` + una tecla: estás grabando una macro | pulsa `q` otra vez para parar |

## Más allá de lo básico

| Orden | Acción |
|---|---|
| `qa` … `q` | grabar una macro en el registro `a` |
| `@a` / `@@` | ejecutar la macro / repetir la última ejecutada |
| `10@a` | ejecutarla 10 veces |
| `"+y` / `"+p` | copiar / pegar con el portapapeles del sistema (si `vim --version` muestra `+clipboard`) |
| `:earlier 10m` / `:later 10m` | devolver el fichero a como estaba hace 10 minutos / deshacer ese salto |

**Neovim** (`nvim`) es una bifurcación moderna de vim con los mismos atajos; todo lo de esta
nota sirve igual (salvo el truco de `sudo tee`).

Muchas herramientas abren el editor por defecto del sistema (`git commit`, `crontab -e`,
`visudo`, `kubectl edit`): si se abre vim sin esperarlo, `:q!` para salir. Cómo cambiarlo, en
[nano](07-nano.md#elegir-el-editor-por-defecto).

## Chuleta

```
MODOS      Esc → normal · i a o → inserción · v V Ctrl+V → visual · : → órdenes
SALIR      :w · :q · :wq · :q! · ZZ
MOVERSE    h j k l · w b e · 0 ^ $ · gg G · :42 · Ctrl+D/U · % · { }
EDITAR     x · dd · yy · p P · u · Ctrl+R · . · >> <<
COMBINAR   d c y  +  w $ iw i" ip G        →  ciw · ci" · dG · yip
BUSCAR     /texto · n N · * · :%s/a/b/gc · :noh
```
