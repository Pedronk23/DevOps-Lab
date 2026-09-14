# SSH — Fundamentos y configuración del servidor

## SSH y OpenSSH

**SSH** (*Secure Shell*): protocolo de red que define cómo se debe establecer una
comunicación segura y cifrada entre dos dispositivos. Se creó para reemplazar protocolos
inseguros como Telnet o rsh.

**OpenSSH** (*Open Secure Shell*): el software, la suite de herramientas de código abierto que
implementa el protocolo SSH. Se creó como una alternativa libre, gratuita y sin restricciones.

> Analogía: SSH son los planos; OpenSSH es el puente real construido siguiendo esos planos.

### Qué incluye OpenSSH

| Componente | Función |
|---|---|
| `ssh` | cliente, para conectarte |
| `sshd` | daemon/servidor que escucha las conexiones |
| `scp` y `sftp` | herramientas de copia de archivos segura |
| `ssh-keygen` y `ssh-agent` | gestores de claves |

En **Windows** se puede usar PuTTY o MobaXterm (además del cliente OpenSSH nativo).

## Cómo funciona la conexión

```
   CLIENTE                                        SERVIDOR (sshd)
      │                                                 │
      │──── 1. versión del protocolo ──────────────────►│
      │◄─── 2. clave de host (host key) ────────────────│
      │        ¿la conozco? → ~/.ssh/known_hosts        │
      │──── 3. intercambio de claves (Diffie-Hellman)──►│
      │        se acuerda una clave de sesión simétrica │
      │════ canal cifrado establecido ══════════════════│
      │──── 4. autenticación (clave pública / pass) ───►│
      │◄─── 5. shell, comando o túnel ──────────────────│
```

- El paso 2 es lo que produce el aviso *"The authenticity of host … can't be established"*:
  la primera vez no conoces la huella del servidor.
- El paso 3 crea una clave **simétrica** de sesión (AES/ChaCha20): el cifrado asimétrico solo
  se usa para negociar, porque es mucho más lento.

## Conexión

```bash
ssh usuario@192.168.1.50
#│      │            └── IP o nombre del servidor
#│      └── usuario que se va a conectar
#└── protocolo/cliente

ssh -p 2222 usuario@servidor        # puerto no estándar
ssh usuario@servidor "uptime"       # ejecutar un comando y salir
ssh -v usuario@servidor             # depurar la negociación
```

Salir de la conexión: `exit` o `Ctrl + D`.

## Autenticación por clave pública

```
   TU PC                                    SERVIDOR
   ~/.ssh/id_ed25519      (privada, nunca sale de aquí)
   ~/.ssh/id_ed25519.pub ─────copias────► ~/.ssh/authorized_keys
                                               │
   el servidor manda un reto ◄─────────────────┘
   lo firmas con la privada ──────────► verifica con la pública
```

```bash
ssh-keygen -t ed25519 -C "pedro@portatil"     # generar par de claves
ssh-copy-id usuario@servidor                  # instalar la pública en el servidor
ssh-keygen -lf ~/.ssh/id_ed25519.pub          # ver la huella
```

| Tipo de clave | Recomendación |
|---|---|
| `ed25519` | la opción por defecto hoy: corta, rápida y segura |
| `rsa` 4096 | válida; usar solo por compatibilidad con sistemas antiguos |
| `dsa`, `rsa` 1024 | obsoletas, deshabilitadas en OpenSSH moderno |

Permisos obligatorios (SSH se niega a funcionar si están mal):

```
~/.ssh              700
~/.ssh/id_ed25519   600
~/.ssh/*.pub        644
~/.ssh/authorized_keys 600
```

## Configuración del cliente: `~/.ssh/config`

Evita escribir la misma línea larga cien veces:

```
Host lab
    HostName 192.168.1.50
    User devops
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    ForwardAgent yes

Host *.interno.local
    User devops
    ProxyJump bastion          # salta a través del bastión

Host *
    ServerAliveInterval 60     # evita que se caiga la sesión por inactividad
    ServerAliveCountMax 3
```

Después basta con `ssh lab`.

## Configuración del servidor: `sshd_config`

```bash
sudo vim /etc/ssh/sshd_config
```

`sshd` es el daemon; este es su archivo de configuración (**lado servidor**).

### Directivas habituales

| Directiva | Qué hace |
|---|---|
| `AddressFamily` | tipo de protocolo que usa para conectarse: `any`, `inet` (IPv4), `inet6` (IPv6) |
| `ListenAddress` | IP en la que escucha para poder entrar |
| `Port` | puerto de escucha (22 por defecto) |
| `Banner` | enlaza un fichero de texto para informar al conectar |
| `PermitRootLogin no` | para que no dejen entrar por root |
| `PasswordAuthentication no` | solo claves, sin contraseñas |
| `PubkeyAuthentication yes` | autenticación por clave pública |
| `MaxAuthTries` | intentos antes de cortar |
| `AllowTcpForwarding` / `AllowAgentForwarding` | permitir túneles y reenvío de agente |
| `ClientAliveInterval` | keepalive desde el servidor |
| `DenyUsers` / `AllowUsers` / `DenyGroups` / `AllowGroups` | listas de control de acceso |

### Endurecimiento mínimo recomendado

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
AllowGroups ssh-users
X11Forwarding no
```

```bash
sudo sshd -t                      # validar la sintaxis ANTES de recargar
sudo systemctl reload sshd
```

> Regla de oro: cuando cambies `sshd_config`, **abre una segunda sesión para probar** antes de
> cerrar la actual. Si te equivocas, aún tienes una puerta abierta.

## Diagnóstico

```bash
ssh -vvv usuario@servidor          # ver toda la negociación
sudo journalctl -u sshd -f         # logs del servidor
sudo ss -tlnp | grep :22           # ¿está escuchando?
ssh-keygen -R servidor             # borrar una host key cambiada
```

| Error | Causa habitual |
|---|---|
| `Permission denied (publickey)` | clave no instalada, permisos mal, o usuario incorrecto |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | reinstalaron el servidor (o MITM): borra la entrada de `known_hosts` |
| `Connection refused` | sshd parado o puerto equivocado |
| `Connection timed out` | firewall o ruta de red |
