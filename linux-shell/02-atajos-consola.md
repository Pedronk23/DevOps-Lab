# Consola — Atajos y comandos rápidos

## Atajos

| Atajo | Acción |
|---|---|
| `Ctrl + D` | salir de la terminal / cerrar sesión |
| `Ctrl + C` | cancelar el comando en ejecución |
| `Ctrl + L` | limpiar la ventana (equivale a `clear`) |
| `Ctrl + R` | buscar comandos en el histórico (pulsando más veces se sigue buscando hacia atrás) |
| `↑` | recuperar el último comando |

> `Ctrl + D` en realidad envía "fin de entrada" (EOF): en una shell vacía la cierra, pero en
> un programa que lee de teclado (`cat`, `python`) termina la entrada.

## Moverse y editar la línea

Funcionan en bash y zsh (modo emacs, el que viene por defecto):

```
   Ctrl+A                Alt+B      Alt+F               Ctrl+E
     │                     │          │                    │
     ▼                     ▼          ▼                    ▼
     kubectl get pods -n  monitoring  -o wide --show-labels
```

| Atajo | Acción |
|---|---|
| `Ctrl + A` | ir al principio de la línea |
| `Ctrl + E` | ir al final de la línea |
| `Alt + B` / `Alt + F` | palabra atrás / adelante |
| `Ctrl + U` | borrar desde el cursor hasta el principio |
| `Ctrl + K` | borrar desde el cursor hasta el final |
| `Ctrl + W` | borrar la palabra anterior |
| `Ctrl + Y` | pegar lo último borrado con `Ctrl+U`/`K`/`W` |
| `Ctrl + _` | deshacer |
| `Alt + .` | inserta el **último argumento** del comando anterior (repetir para ir más atrás) |
| `Ctrl + X`, `Ctrl + E` | abre la línea en `$EDITOR` para editar comandos largos (bash) |
| `Tab` / `Tab Tab` | autocompletar / mostrar opciones |

## Control de procesos

| Atajo / comando | Acción |
|---|---|
| `Ctrl + Z` | suspende el proceso en primer plano (no lo mata) |
| `fg` | lo devuelve al primer plano |
| `bg` | lo continúa en segundo plano |
| `jobs` | lista los procesos suspendidos o en segundo plano de esta shell |
| `comando &` | lanzarlo directamente en segundo plano |
| `nohup comando &` | que sobreviva al cerrar la sesión |
| `disown` | desvincular un job ya lanzado de la shell |
| `Ctrl + S` / `Ctrl + Q` | congela / descongela la salida de la terminal |

> Si la terminal "se ha quedado colgada" sin motivo, prueba `Ctrl + Q`: probablemente
> pulsaste `Ctrl + S` sin querer.

## Historial

| Expansión | Qué hace | Ejemplo |
|---|---|---|
| `!!` | repite el último comando | `sudo !!` → el mismo, con sudo |
| `!$` | último argumento del comando anterior | `mkdir /opt/app && cd !$` |
| `!n` | comando número `n` del `history` | `!512` |
| `!texto` | último comando que empezaba por "texto" | `!ssh` |
| `^viejo^nuevo` | repite el último cambiando un texto | `^staging^production` |

```bash
history | grep kubectl        # buscar
history -d 1042               # borrar una entrada (p. ej. si escribiste una contraseña)
 export TOKEN=abc123          # con espacio delante no se guarda (si HISTCONTROL/HIST_IGNORE_SPACE)
```

## Comandos rápidos

| Comando | Acción |
|---|---|
| `history` | muestra el histórico de comandos |
| `echo $SHELL` | shell activa |
| `cd` | vuelve al directorio raíz del usuario (`$HOME`) |
| `cd ..` | una carpeta atrás |
| `cd -` | vuelve al directorio **anterior** (alterna entre dos) |
| `pwd` | directorio actual |
| `type comando` / `which comando` | qué se ejecuta realmente: alias, función o binario (y su ruta) |
| `man comando` / `comando --help` | ayuda |
| `tldr comando` | ejemplos prácticos resumidos (hay que instalarlo) |
| `watch -n 2 'kubectl get pods'` | repite un comando cada 2 segundos |
| `reset` | arregla la terminal si se ha llenado de caracteres raros (p. ej. tras un `cat` a un binario) |

## Redirecciones y tuberías

| Símbolo | Qué hace |
|---|---|
| `>` | salida estándar a fichero (sobrescribe) |
| `>>` | salida estándar a fichero (añade) |
| `2>` | errores a fichero |
| `2>&1` | errores al mismo sitio que la salida |
| `&>` | salida y errores juntos (bash) |
| `<` | lee la entrada de un fichero |
| `\|` | la salida de uno es la entrada del siguiente |
| `tee` | escribe a fichero **y** a pantalla |
| `/dev/null` | "la papelera": lo que se manda ahí desaparece |

```bash
comando > salida.log 2>&1              # todo a un fichero
comando 2>/dev/null                    # ocultar errores
kubectl get pods -A | tee pods.txt     # ver y guardar
sudo tee /etc/app.conf < app.conf      # escribir en un fichero de root (sudo con > no funciona)
```

> `sudo echo "x" > /etc/fichero` **falla**: la redirección la hace tu shell, sin privilegios,
> antes de que `sudo` entre en juego. Por eso se usa `| sudo tee`.

## Sesiones que sobreviven a una desconexión: tmux

Imprescindible cuando lanzas algo largo por SSH (una actualización, una copia grande).

| Acción | Comando |
|---|---|
| Nueva sesión con nombre | `tmux new -s mantenimiento` |
| Separarse (la sesión sigue viva) | `Ctrl + B`, luego `D` |
| Listar sesiones | `tmux ls` |
| Volver a entrar | `tmux attach -t mantenimiento` |
| Dividir en vertical / horizontal | `Ctrl + B`, `%` / `Ctrl + B`, `"` |
| Moverse entre paneles | `Ctrl + B`, flechas |

```
   SSH se corta ──► la sesión tmux sigue ejecutándose en el servidor
   vuelves a entrar ──► tmux attach ──► todo sigue donde lo dejaste
```
