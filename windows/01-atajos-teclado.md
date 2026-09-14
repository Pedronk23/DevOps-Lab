# Windows — Atajos de teclado

## Movimiento básico

| Atajo | Acción |
|---|---|
| `Tab` | moverse entre elementos |
| `Shift + Tab` | ir hacia atrás |
| `Enter` | abrir / confirmar |
| `Esc` | cancelar / cerrar |
| Flechas | navegar |
| Barra espaciadora | seleccionar |
| `Alt` | activa la barra de menú; muestra las letras de acceso rápido |
| `Shift + F10` | menú contextual (el "clic derecho" del teclado) |

## Ventanas y escritorios

| Atajo | Acción |
|---|---|
| `Alt + Tab` | cambiar entre ventanas |
| `Win + Tab` | vista de tareas |
| `Win + D` | mostrar escritorio |
| `Win + ↑` | maximizar ventana |
| `Win + ↓` | minimizar ventana |
| `Win + ← / →` | ajustar ventana a los lados |
| `Alt + F4` | cerrar ventana |
| `Ctrl + Shift + Esc` | administrador de tareas |
| `Win + Shift + ← / →` | mover la ventana al otro monitor |
| `Win + Home` | minimizar todas menos la activa |
| `Win + número` | abrir o cambiar a la aplicación N de la barra de tareas |
| `Win + P` | modo de proyección (duplicar, extender, solo segunda pantalla) |

### Escritorios virtuales

Muy útil para separar contextos: un escritorio para el lab, otro para correo y documentación.

| Atajo | Acción |
|---|---|
| `Win + Ctrl + D` | crear escritorio virtual |
| `Win + Ctrl + ← / →` | cambiar de escritorio |
| `Win + Ctrl + F4` | cerrar el escritorio actual (las ventanas pasan al contiguo) |

## Buscar y abrir cosas

| Atajo | Acción |
|---|---|
| `Win` | abrir menú inicio |
| `Win + S` | buscar |
| `Win + R` | abrir "Ejecutar" |
| `Win + E` | explorador de archivos |
| `Win + I` | configuración |
| `Win + V` | abrir portapapeles |
| `Win + T` | seleccionar barra de tareas (`← / →` para moverse, `Enter` para abrir) |
| `Win + X` | menú de administración (el del clic derecho en Inicio) |
| `Win + Pausa` | información del sistema (nombre del equipo, RAM, versión) |
| `Win + .` | panel de emojis y símbolos |

> `Win + V` requiere activar el historial del portapapeles la primera vez. Ojo: guarda
> **todo** lo que copias, incluidas contraseñas.

## Ejecutar (`Win + R`): consolas de administración

Escribir el nombre y `Enter`. Con **`Ctrl + Shift + Enter`** se abre **como administrador**.

| Comando | Abre |
|---|---|
| `services.msc` | servicios |
| `eventvwr.msc` | visor de eventos |
| `compmgmt.msc` | administración de equipos (discos, usuarios, servicios, eventos…) |
| `taskschd.msc` | programador de tareas |
| `devmgmt.msc` | administrador de dispositivos |
| `diskmgmt.msc` | administración de discos |
| `lusrmgr.msc` | usuarios y grupos locales |
| `gpedit.msc` | directivas de grupo locales |
| `certlm.msc` / `certmgr.msc` | certificados del equipo / del usuario |
| `wf.msc` | firewall con seguridad avanzada |
| `ncpa.cpl` | conexiones de red (adaptadores) |
| `appwiz.cpl` | programas y características |
| `sysdm.cpl` | propiedades del sistema (nombre, dominio, variables de entorno, escritorio remoto) |
| `regedit` | editor del registro |
| `msinfo32` | información detallada del sistema |
| `resmon` | monitor de recursos (qué proceso usa disco, red, puertos) |
| `perfmon` | monitor de rendimiento |
| `winver` | versión exacta de Windows |
| `mstsc` | escritorio remoto |
| `control` | panel de control clásico |

Con RSAT (herramientas de administración remota) o en un controlador de dominio:

| Comando | Abre |
|---|---|
| `dsa.msc` | usuarios y equipos de Active Directory |
| `gpmc.msc` | administración de directivas de grupo |
| `dnsmgmt.msc` | DNS |
| `dhcpmgmt.msc` | DHCP |

Rutas rápidas:

| Escribir | Abre |
|---|---|
| `%temp%` | temporales del usuario |
| `%appdata%` | datos de aplicación (Roaming) |
| `%programdata%` | datos de aplicación de todos los usuarios |
| `shell:startup` | programas que arrancan al iniciar sesión (usuario) |
| `shell:common startup` | lo mismo, para todos los usuarios |
| `\\servidor\C$` | recurso administrativo remoto (ver [acceso remoto](02-acceso-remoto.md)) |

## Explorador de archivos

| Atajo | Acción |
|---|---|
| `Alt + D` o `Ctrl + L` | ir a la barra de dirección |
| `Alt + ↑` | carpeta superior |
| `Alt + ← / →` | atrás / adelante |
| `Ctrl + Shift + N` | nueva carpeta |
| `F2` | renombrar |
| `Alt + Enter` | propiedades |
| `Shift + Supr` | borrar sin pasar por la papelera |
| `Ctrl + Shift + C` | copiar la ruta del elemento (Windows 11) |
| `Shift + clic derecho` | menú extendido ("Copiar como ruta de acceso", "Abrir PowerShell aquí") |

> Truco: en la barra de dirección escribe `cmd` o `powershell` y pulsa `Enter`: se abre la
> consola **ya situada en esa carpeta**.

## Capturas de pantalla

| Atajo | Acción |
|---|---|
| `Win + Shift + S` | recorte de una zona (al portapapeles) |
| `Win + ImpPnt` | captura completa guardada en `Imágenes\Capturas de pantalla` |
| `Alt + ImpPnt` | solo la ventana activa (al portapapeles) |

## Terminal y sesión

| Atajo | Acción |
|---|---|
| `Win + X`, `A` | terminal como administrador |
| `Win + X`, `I` | terminal como usuario |
| `Ctrl + Alt + Supr` | cambiar contraseña, bloquear, cerrar sesión |
| `Win + L` | bloquear el equipo (hazlo **siempre** al levantarte) |

### Windows Terminal

| Atajo | Acción |
|---|---|
| `Ctrl + Shift + T` | nueva pestaña |
| `Ctrl + Shift + 1…9` | nueva pestaña con el perfil N (PowerShell, CMD, WSL…) |
| `Alt + Shift + D` | dividir el panel duplicando el perfil |
| `Alt + Shift + +` / `Alt + Shift + -` | dividir en vertical / horizontal |
| `Alt + flechas` | moverse entre paneles |
| `Ctrl + Shift + W` | cerrar panel o pestaña |
| `Ctrl + Shift + F` | buscar en la salida |
| `Ctrl + Shift + P` | paleta de comandos |
| `Ctrl + ,` | configuración |
| `Ctrl + Tab` | siguiente pestaña |

## En página web

| Atajo | Acción |
|---|---|
| `Ctrl + Shift + I` | inspeccionar página |
| Icono pantalla + ratón | permite clicar en un elemento y te lleva a él en el inspector |

| Atajo | Acción |
|---|---|
| `F12` | abrir/cerrar herramientas de desarrollo |
| `Ctrl + Shift + C` | activar directamente el selector de elementos (el icono de arriba) |
| `Ctrl + Shift + J` | consola JavaScript (Chrome / Edge) |
| `Ctrl + Shift + R` o `Ctrl + F5` | recarga ignorando la caché |
| `Ctrl + U` | ver código fuente |
| `Ctrl + Shift + M` (con F12 abierto) | simular móvil (Chrome / Edge) |
| `Ctrl + Shift + N` | ventana de incógnito (Chrome / Edge) |

> Para depurar una web o una API desde el navegador, la pestaña **Network** de F12 es la
> reina: códigos HTTP, cabeceras, tiempos y cookies. Marca *Disable cache* y *Preserve log*
> cuando haya redirecciones.
