# Linux — Permisos

Los permisos se reparten en tres bloques: **user**, **group** y **other**.

```
USER      GROUP     OTHER
4 2 1     4 2 1     4 2 1
r w x     r w x     r w x
```

| Letra | Valor | Significado |
|---|---|---|
| `r` | 4 | read (lectura) |
| `w` | 2 | write (escritura) |
| `x` | 1 | executable (ejecución) |

Se suman los valores de cada bloque para formar el número octal.

## Leer `ls -l`

```
-rwxr-x---  1  pedro  devops  4096  sep 14 10:22  deploy.sh
│└┬┘└┬┘└┬┘  │  └─┬─┘  └──┬─┘  └┬─┘  └────┬─────┘  └───┬───┘
│ │  │  │   │    │       │     │        │            └── nombre
│ │  │  │   │    │       │     │        └── última modificación
│ │  │  │   │    │       │     └── tamaño en bytes
│ │  │  │   │    │       └── grupo propietario
│ │  │  │   │    └── usuario propietario
│ │  │  │   └── número de enlaces duros
│ │  │  └── other: ---  (nada)
│ │  └── group: r-x     (leer y ejecutar)
│ └── user:  rwx        (todo)
└── tipo de fichero
```

| Tipo | Significado |
|---|---|
| `-` | fichero normal |
| `d` | directorio |
| `l` | enlace simbólico |
| `c` / `b` | dispositivo de caracteres / bloques (`/dev/tty`, `/dev/sda`) |
| `s` | socket (`/var/run/docker.sock`) |
| `p` | tubería con nombre (FIFO) |

## Ejemplos

| Octal | Equivalente | Lectura |
|---|---|---|
| `600` | `rw- --- ---` | el dueño lee y escribe; nadie más accede |
| `660` | `rw- rw- ---` | dueño y grupo leen y escriben |
| `644` | `rw- r-- r--` | dueño escribe; el resto solo lee |
| `755` | `rwx r-x r-x` | típico de directorios y binarios |
| `700` | `rwx --- ---` | directorio privado (`~/.ssh`) |
| `750` | `rwx r-x ---` | el grupo entra, el resto no |

```bash
chmod 600 fichero      # cambiar permisos
chmod 755 script.sh
chown usuario:grupo fichero   # cambiar propietario
ls -l                  # ver permisos
```

> Regla práctica: claves privadas y ficheros de credenciales, siempre `600`.

## En directorios, `rwx` significa otra cosa

| Permiso | En un fichero | En un directorio |
|---|---|---|
| `r` | leer el contenido | **listar** los nombres (`ls`) |
| `w` | modificar el contenido | **crear, borrar y renombrar** ficheros dentro |
| `x` | ejecutarlo | **atravesarlo**: entrar (`cd`) y acceder a lo que hay dentro |

Consecuencias que sorprenden:

- Para leer `/opt/app/config.yml` necesitas `x` en `/opt` y en `/opt/app`, además de `r` en
  el fichero.
- Quien tiene `w` en un directorio **puede borrar un fichero aunque no tenga permisos sobre
  él**. Por eso existe el *sticky bit* (ver abajo).
- Un directorio `r--` sin `x` deja ver los nombres pero no abrir nada.

## Notación simbólica

```
chmod  [ugoa] [+-=] [rwxXst]  fichero
        │      │     └── permisos
        │      └── añadir, quitar, fijar exactamente
        └── user, group, other, all
```

```bash
chmod u+x script.sh          # añade ejecución al dueño
chmod g-w informe.txt        # quita escritura al grupo
chmod o= secreto.txt         # other sin ningún permiso
chmod a+r publico.txt        # todos pueden leer
chmod u=rw,go=r fichero      # equivale a 644
chmod -R g+rX /srv/compartido   # X mayúscula: x solo en directorios (y ficheros ya ejecutables)
```

> `chmod -R 755` sobre un árbol hace **ejecutables todos los ficheros**. Con `-R` usa `X`
> mayúscula, o separa directorios y ficheros:
> `find /srv/web -type d -exec chmod 755 {} +` y `find /srv/web -type f -exec chmod 644 {} +`

## Usuarios y grupos

```bash
id                               # uid, gid y grupos del usuario actual
id devops                        # de otro usuario
groups                           # solo los grupos
sudo usermod -aG docker devops   # añadir a un grupo (-a es vital: sin él REEMPLAZA los grupos)
sudo chown -R devops:devops /opt/app
sudo chgrp devops informe.txt
getent group docker              # miembros de un grupo
```

> Un usuario añadido a un grupo **no obtiene el permiso hasta que vuelve a iniciar sesión**
> (o ejecuta `newgrp docker` en la shell actual). Clásico con el grupo `docker`.

## umask: permisos por defecto al crear

El `umask` indica qué permisos se **quitan** a los nuevos ficheros y directorios.

| umask | Ficheros nuevos | Directorios nuevos | Uso |
|---|---|---|---|
| `022` | `644` | `755` | lo habitual |
| `002` | `664` | `775` | trabajo en grupo |
| `077` | `600` | `700` | usuarios de servicio, datos sensibles |

```bash
umask          # ver el actual
umask 077      # para esta shell
```

## Bits especiales

| Bit | Octal | En ficheros | En directorios | Se ve como |
|---|---|---|---|---|
| **SUID** | `4000` | se ejecuta con los permisos del **dueño** | — | `rws` en user |
| **SGID** | `2000` | se ejecuta con los permisos del **grupo** | los ficheros creados heredan el grupo del directorio | `rws` en group |
| **Sticky** | `1000` | — | solo el dueño de un fichero puede borrarlo | `rwt` en other |

```bash
ls -l /usr/bin/passwd     # -rwsr-xr-x → SUID: un usuario normal puede cambiar /etc/shadow
ls -ld /tmp               # drwxrwxrwt → sticky: todos escriben, nadie borra lo ajeno

chmod 2775 /srv/proyecto  # SGID en carpeta compartida: todo lo nuevo pertenece al grupo
chmod +t /srv/subidas     # sticky bit

# auditoría: binarios con SUID en el sistema (vector clásico de escalada de privilegios)
sudo find / -perm -4000 -type f 2>/dev/null
```

## ACLs: cuando user/group/other se queda corto

Permiten dar permisos a usuarios o grupos concretos adicionales.

```bash
setfacl -m u:ansible:rx /opt/app              # un usuario más con lectura y ejecución
setfacl -m g:auditores:r /var/log/app.log     # un grupo más
setfacl -d -m g:devops:rwX /srv/compartido    # ACL por defecto: la heredan los ficheros nuevos
getfacl /opt/app                              # ver ACLs
setfacl -b /opt/app                           # quitar todas
```

Un `+` al final de los permisos en `ls -l` (`drwxr-x---+`) indica que hay ACLs.

## sudo

```bash
sudo -l                        # qué puedo ejecutar con sudo
sudo visudo                    # editar /etc/sudoers con validación de sintaxis
sudo visudo -f /etc/sudoers.d/devops
```

```
# /etc/sudoers.d/devops
%devops   ALL=(ALL) ALL                                   # el grupo devops puede todo (con contraseña)
ansible   ALL=(ALL) NOPASSWD: ALL                         # usuario de automatización
monitor   ALL=(root) NOPASSWD: /usr/bin/systemctl status *  # solo un comando concreto
```

> Edita siempre con `visudo`: un error de sintaxis en sudoers te deja **sin sudo**.

## Permisos recomendados

| Ruta | Permisos | Dueño |
|---|---|---|
| `~/.ssh` | `700` | usuario |
| `~/.ssh/id_ed25519`, `authorized_keys` | `600` | usuario |
| `~/.kube/config` | `600` | usuario |
| Scripts propios | `750` o `755` | usuario |
| `/etc/shadow` | `640` (Debian, grupo `shadow`) / `000` (RHEL) | root |
| Web estática | directorios `755`, ficheros `644` | usuario de despliegue |
| Ficheros `.env` con credenciales | `600` o `640` | usuario del servicio |

## Otros comandos útiles

```bash
stat -c '%a %U:%G %n' fichero     # permisos en octal, dueño y nombre
namei -l /opt/app/config.yml      # permisos de CADA directorio del camino (diagnóstico rápido)
sudo chattr +i /etc/resolv.conf   # inmutable: ni root puede modificarlo hasta chattr -i
lsattr /etc/resolv.conf
```

## Errores típicos

| Síntoma | Causa habitual |
|---|---|
| `Permission denied` al ejecutar `./script.sh` | falta `x`, o el sistema de ficheros está montado con `noexec` (pasa en `/tmp`) |
| `Permission denied` leyendo un fichero con `r` | falta `x` en algún directorio del camino: `namei -l` |
| `docker: permission denied … docker.sock` | usuario sin el grupo `docker`, o no ha vuelto a iniciar sesión |
| SSH ignora la clave | permisos demasiado abiertos en `~/.ssh` o en el home |
| "Lo arreglé con `chmod 777`" | no está arreglado: está abierto a todos. Buscar el dueño/grupo correcto |
