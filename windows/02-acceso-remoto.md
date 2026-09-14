# Windows — Acceso remoto

## Opciones de un vistazo

| Necesito… | Herramienta | Puerto |
|---|---|---|
| El escritorio gráfico | RDP (`mstsc`) | 3389 |
| Ficheros | recursos compartidos SMB (`\\servidor\C$`) | 445 |
| Una consola para ejecutar comandos | PowerShell Remoting / WinRM | 5985 / 5986 |
| Consola por SSH | OpenSSH Server de Windows | 22 |
| Automatizar | Ansible (sobre WinRM o SSH) | 5985 / 5986 / 22 |

## RDP (Remote Desktop Protocol)

Tecnología que usa Windows para conectarse al escritorio de otro ordenador.

```cmd
mstsc
```

Se ejecuta desde `CMD` o desde `Win + R` y pide el host al que conectar.
Puerto por defecto: **3389/TCP**.

### Opciones de `mstsc`

| Opción | Qué hace |
|---|---|
| `mstsc /v:srv01.lab.local` | conectar directamente |
| `mstsc /v:srv01:13389` | puerto no estándar (o túnel local) |
| `/admin` | entra en la sesión de administración (útil si el servidor dice que no hay licencias) |
| `/f` | pantalla completa |
| `/w:1600 /h:900` | tamaño de ventana |
| `/multimon` | usar todos los monitores |
| `mstsc archivo.rdp` | abrir una conexión guardada (se guarda desde "Mostrar opciones") |

### Atajos dentro de una sesión RDP

| Atajo | Equivale a |
|---|---|
| `Ctrl + Alt + Fin` | `Ctrl + Alt + Supr` en el equipo **remoto** |
| `Ctrl + Alt + Pausa` | alternar ventana / pantalla completa |
| `Alt + Re Pág` / `Alt + Av Pág` | `Alt + Tab` en el remoto (en ventana) |

### Habilitar RDP en un servidor

```powershell
# permitir conexiones
Set-ItemProperty 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name fDenyTSConnections -Value 0

# exigir NLA (autenticación antes de mostrar la pantalla de login)
Set-ItemProperty 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name UserAuthentication -Value 1

# abrir el firewall (por nombre interno: no depende del idioma del sistema)
Enable-NetFirewallRule -Name 'RemoteDesktop-UserMode-In-TCP', 'RemoteDesktop-UserMode-In-UDP'

# dar acceso a un usuario no administrador (S-1-5-32-555 = "Usuarios de escritorio remoto")
Add-LocalGroupMember -SID 'S-1-5-32-555' -Member 'LAB\pedro'
```

> Muchos ejemplos de Internet usan `-DisplayGroup "Remote Desktop"`, que falla en un Windows
> en español (el grupo se llama "Escritorio remoto"). Los `-Name` internos y los SID no se
> traducen.

### Sesiones: el clásico "demasiados usuarios conectados"

Windows Server permite **2 sesiones administrativas simultáneas** sin licencias de RDS.
Si alguien cerró la ventana sin cerrar sesión, la sesión sigue ocupando hueco:

```cmd
quser /server:srv01                 :: listar sesiones (ID, estado, tiempo inactivo)
logoff 2 /server:srv01              :: expulsar la sesión con ID 2
```

```
 USERNAME   SESSIONNAME   ID  STATE   IDLE TIME  LOGON TIME
 admin-local               2  Disc      3+02:11  10/09/2026 9:14
 pedro      rdp-tcp#12     3  Active          .  14/09/2026 8:30
```

> Para salir de verdad: **Cerrar sesión**, no la X de la ventana (eso solo *desconecta* y los
> procesos siguen corriendo).

### Seguridad de RDP

- **No exponer 3389 a Internet**: es uno de los puertos más atacados (fuerza bruta y
  vulnerabilidades como BlueKeep).
- Acceder por VPN, **RD Gateway** (RDP encapsulado en HTTPS 443) o un bastión.
- NLA activado siempre.
- Un túnel SSH también sirve (ver [túneles SSH](../seguridad/06-tuneles-ssh-x11-socks.md)):

```bash
ssh -L 13389:srv01.lab.local:3389 devops@bastion
# y en Windows:  mstsc /v:localhost:13389
```

## Entrar en un servidor desde el explorador de archivos

En la barra de ruta del explorador se escribe el recurso administrativo del disco:

```
\\srv01.lab.local\C$
```

Pedirá credenciales; el usuario se indica con el prefijo `.\` para que sea una
cuenta **local** del servidor y no del dominio:

```
.\admin-local  + contraseña
```

`C$` es un recurso compartido administrativo oculto: da acceso a la raíz de `C:`
y requiere permisos de administrador en la máquina destino.

### Formatos de usuario

| Formato | Significa |
|---|---|
| `.\admin-local` | cuenta **local** del equipo al que te conectas |
| `SRV01\admin-local` | lo mismo, con el nombre del equipo explícito |
| `LAB\pedro` | cuenta del **dominio** LAB (nombre NetBIOS) |
| `pedro@lab.local` | cuenta del dominio en formato UPN |

### Recursos administrativos ocultos

El `$` final hace que no aparezcan al explorar `\\servidor`.

| Recurso | Apunta a |
|---|---|
| `C$`, `D$`… | raíz de cada disco |
| `ADMIN$` | `C:\Windows` |
| `IPC$` | canal de comunicación entre procesos (lo usan RPC y muchas herramientas de administración) |

### Desde la consola: `net use`

```cmd
net use Z: \\srv01\C$ /user:SRV01\admin-local *        :: el * pide la contraseña sin mostrarla
net use                                            :: conexiones actuales
net use Z: /delete
net use * /delete /y                               :: borrar todas
```

```powershell
Get-SmbMapping                                     # equivalente en PowerShell
New-SmbMapping -LocalPath Z: -RemotePath \\srv01\despliegues
```

### Crear un recurso compartido

```powershell
New-SmbShare -Name despliegues -Path D:\despliegues -ChangeAccess 'LAB\devops' -ReadAccess 'LAB\auditores'
Get-SmbShare
Get-SmbShareAccess -Name despliegues
```

> Hay **dos capas** de permisos: los del recurso compartido y los NTFS de la carpeta. Se
> aplica **el más restrictivo** de los dos.

## Errores típicos

| Error | Causa / solución |
|---|---|
| `Multiple connections to a server or shared resource by the same user, using more than one user name, are not allowed` (1219) | ya hay una conexión a ese servidor con otro usuario: `net use \\srv01\IPC$ /delete` (o `net use * /delete`) y reintentar |
| `Access is denied` a `C$` con una cuenta **local** administradora | UAC remoto filtra el token de los administradores locales (salvo la cuenta `Administrator` integrada). Usar cuenta de dominio, o poner `LocalAccountTokenFilterPolicy = 1` en `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` (**reduce la seguridad**: solo si hace falta) |
| `The network path was not found` | nombre mal, puerto 445 bloqueado o el servicio "Servidor" parado |
| RDP: `The remote computer requires Network Level Authentication` | el cliente no puede hacer NLA (dominio inaccesible, contraseña caducada) |
| RDP: `An authentication error has occurred… CredSSP encryption oracle remediation` | cliente y servidor con parches de seguridad distintos: actualizar el lado antiguo |
| RDP: `The number of connections to this computer is limited` | sesiones desconectadas ocupando hueco: `quser` + `logoff`, o `mstsc /admin` |

## Diagnóstico rápido

```powershell
Test-NetConnection srv01 -Port 3389          # RDP
Test-NetConnection srv01 -Port 445           # SMB
Test-NetConnection srv01 -Port 5985          # WinRM
Get-SmbServerConfiguration | Select-Object EnableSMB1Protocol, EnableSMB2Protocol   # SMBv1 debe estar a False
```
