# SSH — Transferencia de archivos: SCP y SFTP

## Panorama

```
        FTP              FTPS             SCP / SFTP
   ┌──────────┐     ┌──────────┐      ┌──────────────┐
   │ sin cifrar│     │ FTP+TLS  │      │ sobre SSH    │
   │ puerto 21 │     │ 990/21   │      │ puerto 22    │
   │ años 70   │     │ parche   │      │ estándar hoy │
   └──────────┘     └──────────┘      └──────────────┘
     evitar           aceptable            usar
```

| Protocolo | Notas |
|---|---|
| **FTP** (File Transfer Protocol) | antiguo, sin seguridad (años 70) |
| **SCP** (Secure Copy Protocol) | usa SSH para empaquetar y cifrar todo |
| **SFTP** (SSH File Transfer Protocol) | nada que ver con FTP: ofrece la seguridad de SSH y un gestor de archivos |

## SCP

- Utiliza el protocolo SSH para empaquetar y cifrar todo.
- **Ventaja**: extremadamente rápido. Solo enviar y recibir.
- **Desventaja**: no es interactivo. Dispara y olvida.
- Se usa para scripts de automatización, copias de seguridad, etc.

```bash
scp archivo.txt usuario@192.168.1.50:/ruta/destino/     # subir
scp usuario@192.168.1.50:/ruta/archivo.txt .            # bajar
scp -r carpeta/ usuario@servidor:/opt/                  # recursivo
scp -P 2222 archivo usuario@servidor:/tmp/              # puerto (P mayúscula)
scp -i ~/.ssh/id_ed25519 archivo usuario@servidor:/tmp/ # clave concreta
scp -C archivo usuario@servidor:/tmp/                   # comprimir en tránsito
```

> Nota: OpenSSH moderno implementa `scp` **por debajo con el protocolo SFTP**, porque el
> protocolo scp original tenía problemas de seguridad. El comando sigue igual.

## SFTP

- Ofrece la seguridad de SSH más un gestor de archivos.
- **Ventaja**: sesión interactiva. Puedes ver carpetas, directorios, borrar, etc.
- Es el estándar en la industria para interactuar con servidores.

```bash
sftp 192.168.1.50      # el servidor pide tu clave
sftp>                  # ya estás en un "túnel" directo al servidor
```

### Comandos dentro de `sftp>`

```
   comandos LOCALES        comandos REMOTOS
   ─────────────────       ─────────────────
   lpwd  lcd  lls          pwd  cd  ls
                           mkdir  rm  rmdir  rename
                           chmod  chown
   put  ← subir            get  ← bajar
```

| Comando | Qué hace |
|---|---|
| `pwd` | ver dónde está el servidor remoto |
| `lpwd` | ver el directorio local |
| `get fichero` | descargar un archivo del servidor a tu PC |
| `put fichero` | subir un archivo de tu PC al servidor |
| `get -r carpeta` | descargar recursivo |
| `ls` / `lls` | listar remoto / local |
| `exit` o `bye` | salir |

Modo no interactivo (para scripts):

```bash
sftp usuario@servidor <<'EOC'
cd /opt/app
put build.tar.gz
bye
EOC
```

## rsync: la opción que falta en el apunte

Para sincronizar directorios grandes es mucho mejor que scp, porque solo transfiere las
diferencias y puede reanudar.

```bash
rsync -avz --progress carpeta/ usuario@servidor:/opt/carpeta/
rsync -avz --delete --dry-run origen/ destino/      # simular y espejar
rsync -avz -e "ssh -p 2222" origen/ usuario@servidor:/opt/
```

| Flag | Significado |
|---|---|
| `-a` | archivo: recursivo, preserva permisos, tiempos, enlaces |
| `-v` | verboso |
| `-z` | comprime en tránsito |
| `--delete` | borra en destino lo que no existe en origen |
| `--dry-run` | simula |

> Cuidado con la barra final: `origen/` copia el **contenido**; `origen` copia la **carpeta**.

## Cuál usar

```
   ¿un fichero suelto en un script?        → scp
   ¿navegar y elegir a mano?               → sftp
   ¿sincronizar carpetas / backups?        → rsync
   ¿interfaz gráfica en Windows?           → WinSCP
```
