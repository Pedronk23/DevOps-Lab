# Linux — GRUB

**GRUB** (*GRand Unified Bootloader*) es el gestor de arranque por defecto en casi todas las
distribuciones de Linux (Ubuntu, Debian, Fedora, Red Hat).

- Es el **primer programa que se ejecuta** al encender el ordenador, justo después de que la
  placa base (BIOS o UEFI) termine sus comprobaciones iniciales.
- Su trabajo principal es **cargar el sistema operativo en la memoria RAM**.

## Secuencia de arranque

```
   ┌──────────────┐   ┌──────────────┐   ┌─────────────────────┐   ┌──────────────┐
   │ BIOS / UEFI  │──►│    GRUB      │──►│ kernel + initramfs  │──►│   systemd    │
   │ POST, busca  │   │ menú, lee    │   │ detecta hardware,   │   │ (PID 1)      │
   │ disco de     │   │ grub.cfg,    │   │ monta la raíz /     │   │ arranca los  │
   │ arranque     │   │ carga kernel │   │                     │   │ servicios    │
   └──────────────┘   └──────────────┘   └─────────────────────┘   └──────┬───────┘
                                                                          ▼
                                                     multi-user.target / graphical.target
```

| Pieza | Qué es |
|---|---|
| **BIOS** (legado) | busca el código de arranque en el MBR del disco |
| **UEFI** (actual) | carga un ejecutable `.efi` de la partición EFI (`/boot/efi`) |
| **kernel** | `/boot/vmlinuz-<versión>` |
| **initramfs** | `/boot/initrd.img-<versión>` o `initramfs-<versión>.img`: mini sistema con los drivers necesarios para montar el disco real |

## Funciones principales

| Función | Detalle |
|---|---|
| **Menú de selección** | si tienes más de un sistema operativo, es esa pantalla negra con letras blancas donde eliges |
| **Cargar diferentes versiones de Linux** | al actualizar el kernel, el sistema guarda la versión anterior por seguridad |
| **Modo de recuperación** | permite arrancar el sistema en modo especial (*recovery*) |

> Si una actualización de kernel rompe el arranque, en el menú de GRUB (*Advanced options*)
> se puede elegir el kernel anterior y arreglarlo desde un sistema que funciona.

## ¿Dónde se configura?

1. Modificas el archivo de texto simple:

```
/etc/default/grub
```

2. Aplicas los cambios con:

```bash
sudo update-grub
# o grub2-mkconfig en algunas distribuciones
```

### Ficheros implicados

| Ruta | Qué es | ¿Se edita? |
|---|---|---|
| `/etc/default/grub` | opciones generales | **sí** |
| `/etc/grub.d/` | scripts que generan cada parte del menú | solo para entradas personalizadas (`40_custom`) |
| `/boot/grub/grub.cfg` (Debian/Ubuntu) | menú final generado | **no**: se sobrescribe |
| `/boot/grub2/grub.cfg` (RHEL/Rocky/Fedora) | menú final generado | **no** |

### Regenerar según la distribución

| Distribución | Comando |
|---|---|
| Debian / Ubuntu | `sudo update-grub` |
| RHEL / Rocky / Alma / Fedora | `sudo grub2-mkconfig -o /boot/grub2/grub.cfg` |
| RHEL 8+ (cambiar argumentos del kernel) | `sudo grubby --update-kernel=ALL --args="…"` |

> En RHEL 8+ las entradas del menú son ficheros BLS en `/boot/loader/entries/`. Para los
> argumentos del kernel, `grubby` es la herramienta recomendada.

## Variables de `/etc/default/grub`

```bash
GRUB_DEFAULT=0                          # entrada por defecto (0 = la primera, o "saved")
GRUB_TIMEOUT=5                          # segundos que se muestra el menú
GRUB_TIMEOUT_STYLE=menu                 # menu | countdown | hidden
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"   # argumentos solo para el arranque normal
GRUB_CMDLINE_LINUX=""                   # argumentos para TODAS las entradas (también recovery)
GRUB_DISABLE_OS_PROBER=true             # no buscar otros sistemas operativos
```

| Variable | Para qué |
|---|---|
| `GRUB_DEFAULT` | qué entrada arranca sola; `saved` recuerda la última elegida |
| `GRUB_TIMEOUT` | en servidores, un valor > 0 permite intervenir por consola remota (iLO/iDRAC) |
| `GRUB_CMDLINE_LINUX` | parámetros del kernel permanentes |
| `GRUB_TERMINAL` / `GRUB_SERIAL_COMMAND` | menú por puerto serie (VMs y servidores sin pantalla) |

## Parámetros del kernel habituales

| Parámetro | Efecto |
|---|---|
| `quiet splash` | arranque sin texto, con pantalla gráfica |
| `nomodeset` | no cargar drivers de vídeo del kernel (pantalla negra tras instalar) |
| `console=ttyS0,115200` | salida por consola serie (VMs en la nube, Proxmox) |
| `systemd.unit=rescue.target` | modo rescate: monousuario con servicios mínimos |
| `systemd.unit=emergency.target` | aún más mínimo: raíz en solo lectura |
| `rd.break` | (RHEL) para dentro del initramfs, antes de montar la raíz real |
| `ipv6.disable=1` | desactivar IPv6 |
| `net.ifnames=0` | nombres de interfaz clásicos (`eth0` en vez de `ens18`) |

```bash
cat /proc/cmdline         # con qué parámetros arrancó el kernel actual
uname -r                  # versión del kernel en uso
ls /boot                  # kernels instalados
```

## Editar el arranque sobre la marcha

1. En el menú de GRUB, sitúate en la entrada y pulsa **`e`**.
2. Busca la línea que empieza por `linux` y añade el parámetro al final.
3. Arranca con **`Ctrl + X`** o **`F10`**.

El cambio es **solo para ese arranque**: no toca ningún fichero.

> ¿No aparece el menú? Mantén **`Shift`** (BIOS) o pulsa **`Esc`** repetidamente (UEFI)
> justo después del logo del fabricante.

## Recuperar la contraseña de root

**Debian / Ubuntu**

```
1. En GRUB: e → en la línea "linux" añadir:   rw init=/bin/bash
2. Ctrl+X → aparece una shell de root
3. passwd root
4. exec /sbin/init        (o reiniciar forzado)
```

**RHEL / Rocky** (con SELinux)

```
1. En GRUB: e → en la línea "linux" añadir:   rd.break
2. Ctrl+X
3. mount -o remount,rw /sysroot
4. chroot /sysroot
5. passwd root
6. touch /.autorelabel     ← imprescindible con SELinux, si no, no podrás entrar
7. exit ; exit
```

> Esto demuestra que **quien tiene acceso a la consola tiene root**. En entornos donde eso
> importa: contraseña en GRUB (`grub-mkpasswd-pbkdf2`) y en la BIOS/UEFI, y cifrado de disco.

## Elegir kernel por defecto

```bash
# Debian/Ubuntu (requiere GRUB_DEFAULT=saved)
grep -E "menuentry '|submenu '" /boot/grub/grub.cfg | cut -d"'" -f2   # ver entradas
sudo grub-set-default "Advanced options for Ubuntu>Ubuntu, with Linux 6.8.0-45-generic"
sudo grub-reboot 1        # arrancar esa entrada solo en el PRÓXIMO reinicio

# RHEL/Rocky
sudo grubby --info=ALL | grep -E '^(index|kernel)'
sudo grubby --set-default /boot/vmlinuz-5.14.0-427.el9.x86_64
```

## Problemas típicos

| Síntoma | Causa / solución |
|---|---|
| Prompt `grub rescue>` | GRUB no encuentra su configuración o su partición (disco cambiado, partición borrada). Arrancar un live, `chroot` y `grub-install` + `update-grub` |
| Prompt `grub>` | encuentra GRUB pero no `grub.cfg`: regenerarlo |
| Arranca y se queda en *emergency mode* | normalmente una línea de `/etc/fstab` que apunta a un disco que no existe (añadir `nofail`) |
| Tras actualizar, pantalla negra | probar el kernel anterior o `nomodeset` |
| Cambié `/etc/default/grub` y no pasa nada | faltó `update-grub` / `grub2-mkconfig` |

```bash
# reinstalar GRUB desde un sistema live (BIOS, disco /dev/sda, raíz en /dev/sda2)
sudo mount /dev/sda2 /mnt
for d in dev proc sys; do sudo mount --bind /$d /mnt/$d; done
sudo chroot /mnt
grub-install /dev/sda
update-grub
```
