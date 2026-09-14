# SSH — SSHFS y VS Code Remote

## SSHFS (Secure Shell Filesystem)

Herramienta que permite **montar un directorio remoto en tu computadora local** utilizando
únicamente una conexión SSH.

```
   TU PC                                    SERVIDOR
   ~/montaje/  ◄══════ FUSE + SSH ═════════ /opt/app
        │
   tus programas locales (VS Code, nautilus, grep…)
   ven una carpeta normal
```

### Ventajas

- **Seguridad nativa**: todo el tráfico de archivos viaja cifrado.
- **Sin configuraciones complejas**: si tienes acceso SSH, ya tienes SSHFS.
- **Comodidad**: puedes usar tus programas locales para modificar los archivos que están en el
  servidor en tiempo real.

### Uso

```bash
sudo apt install sshfs
mkdir -p ~/montaje

sshfs usuario@192.168.1.50:/opt/app ~/montaje
fusermount -u ~/montaje                    # desmontar (Linux)
umount ~/montaje                           # macOS
```

Opciones útiles para que no se cuelgue con cortes de red:

```bash
sshfs usuario@servidor:/opt/app ~/montaje \
  -o reconnect,ServerAliveInterval=15,ServerAliveCountMax=3,follow_symlinks
```

Montaje permanente en `/etc/fstab`:

```
usuario@servidor:/opt/app  /home/pedro/montaje  fuse.sshfs  noauto,x-systemd.automount,_netdev,reconnect,IdentityFile=/home/pedro/.ssh/id_ed25519  0 0
```

### Limitaciones

- **Latencia**: cada operación de fichero es una ida y vuelta por la red. Compilar o hacer
  `grep -r` sobre un montaje SSHFS es lentísimo.
- Si se corta la red, la carpeta se queda "colgada" hasta el timeout.
- No es apto para bases de datos ni para cargas con muchas escrituras pequeñas.

## Trabajar sobre Linux desde Windows con VS Code

1. VS Code → extensión **Remote - SSH**.
2. `Ctrl + Shift + P` → *ssh*.
3. *Add New SSH Host* → `ssh usuario@192.168.1.50`.
4. Se abre una **nueva ventana** ya conectada al Linux remoto.

### Por qué es mejor que SSHFS

```
        SSHFS                          VS Code Remote - SSH
   ┌──────────────┐               ┌──────────────┐
   │ VS Code      │               │ VS Code UI   │  ← solo la interfaz local
   │ (local)      │               └──────┬───────┘
   └──────┬───────┘                      │ SSH
          │ cada lectura va por red      ▼
          ▼                       ┌──────────────┐
   ficheros remotos               │ vscode-server│  ← extensiones, terminal,
                                  │ (en el host) │     indexado y búsqueda
                                  └──────────────┘     se ejecutan allí
   lento con proyectos grandes     rápido: solo viaja la UI
```

La extensión instala un pequeño servidor en el host remoto, así que el terminal integrado,
las extensiones y la búsqueda se ejecutan **en el servidor**, no sobre la red.

### Requisitos y detalles

- El host remoto necesita una arquitectura soportada (x86_64/arm64) y `curl`/`wget`.
- Reutiliza tu `~/.ssh/config`: si tienes un `Host lab`, aparece en la lista directamente.
- Funciona con `ProxyJump`, así que puedes editar en máquinas detrás de un bastión.
- Alternativa equivalente para contenedores: *Dev Containers*; y para Kubernetes,
  `kubectl exec` + la extensión de Kubernetes.
