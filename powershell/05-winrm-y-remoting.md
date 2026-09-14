# PowerShell — WinRM y remoting

## Qué es WinRM

**WinRM** (*Windows Remote Management*) es la implementación de Microsoft del protocolo
**WS-Management**: peticiones SOAP sobre HTTP(S). Es lo que usa PowerShell Remoting y lo que
usa Ansible para gestionar Windows.

```
   TU EQUIPO                                          SERVIDOR
   Enter-PSSession / Invoke-Command                   servicio WinRM
          │                                                │
          │──── HTTP  5985  (cifrado a nivel de mensaje    │
          │                  con Kerberos/NTLM) ──────────►│──► wsmprovhost.exe
          │──── HTTPS 5986  (TLS) ────────────────────────►│    (una shell PowerShell
          │                                                │     por sesión)
```

| Puerto | Transporte | Notas |
|---|---|---|
| **5985** | HTTP | por defecto; en dominio el contenido va cifrado por Kerberos igualmente |
| **5986** | HTTPS | necesario para Basic auth y recomendable fuera de dominio |

## Habilitar y comprobar

```powershell
# en el servidor (como administrador)
Enable-PSRemoting -Force           # arranca WinRM, crea listener HTTP y regla de firewall
winrm quickconfig                  # equivalente clásico

# desde el cliente
Test-WSMan srv01                   # ¿responde WinRM?
Test-NetConnection srv01 -Port 5985
```

```cmd
winrm enumerate winrm/config/listener
winrm get winrm/config
```

> En Windows Server el remoting ya viene activado. En Windows cliente (10/11) no, y
> `Enable-PSRemoting` falla si alguna red está como **Pública**
> (`-SkipNetworkProfileCheck` o cambiar el perfil de red).

## Guardar y cargar credenciales

Primero se generan y se exportan a XML (quedan cifradas para el usuario y equipo actuales):

```powershell
Get-Credential | Export-Clixml $ENV:HOMEPATH\creds\admin-local.xml
```

Después se cargan en una variable cuando se necesiten:

```powershell
$c = Import-Clixml $ENV:HOMEPATH\creds\admin-local.xml
```

> El cifrado usa **DPAPI**: solo **ese usuario en ese equipo** puede descifrar el XML.
> Si copias el fichero a otra máquina o lo usa otra cuenta (p. ej. la de una tarea
> programada), no funciona. Para eso: `Microsoft.PowerShell.SecretManagement` o un gestor de
> secretos.

## Entrar en la sesión remota

Forma directa:

```powershell
Enter-PSSession -ComputerName "srv01" -Credential $c
```

Con **splatting** (más legible y reutilizable): se define una hashtable con los parámetros
de conexión y se usa `@` al llamar.

```powershell
$enter = @{
    ComputerName = "srv01"
    Credential   = $c
}

Enter-PSSession @enter
```

- `Enter-PSSession @enter` → entra directamente en el servidor.
- `exsn` → alias de `Exit-PSSession`: salir del servidor.

El prompt cambia para recordarte dónde estás:

```
[srv01]: PS C:\Users\admin-local\Documents>
```

## Tres formas de trabajar en remoto

| Forma | Cmdlet | Cuándo |
|---|---|---|
| **Interactiva 1:1** | `Enter-PSSession` | explorar y diagnosticar a mano |
| **Uno a muchos** | `Invoke-Command -ComputerName a,b,c` | lanzar lo mismo en N servidores en paralelo |
| **Sesión persistente** | `New-PSSession` + `Invoke-Command -Session` | varias órdenes seguidas manteniendo estado, copiar ficheros |

### Invoke-Command: uno a muchos

```powershell
$servidores = 'web01', 'web02', 'web03'

Invoke-Command -ComputerName $servidores -Credential $c -ScriptBlock {
    Get-Service W3SVC | Select-Object Status
}
# cada resultado lleva la propiedad PSComputerName con el servidor que lo generó
```

- Se ejecuta **en paralelo** (32 conexiones simultáneas por defecto, `-ThrottleLimit`).
- Lo que devuelve son objetos **deserializados**: tienen las propiedades, pero no los métodos.

### Pasar variables locales: `$using:`

El scriptblock se ejecuta en el servidor y **no ve tus variables**:

```powershell
$servicio = 'W3SVC'

Invoke-Command -ComputerName web01 -ScriptBlock { Restart-Service $servicio }        # ✘ $servicio vacío
Invoke-Command -ComputerName web01 -ScriptBlock { Restart-Service $using:servicio }  # ✔
```

### Sesiones persistentes y copia de ficheros

```powershell
$s = New-PSSession -ComputerName web01 -Credential $c

Invoke-Command -Session $s -ScriptBlock { $inicio = Get-Date }
Invoke-Command -Session $s -ScriptBlock { (Get-Date) - $inicio }    # la variable sigue ahí

Copy-Item .\app.war -Destination 'C:\apps\MiAppWeb\Tomcat 9\webapps\' -ToSession $s
Copy-Item 'C:\apps\MiAppWeb\Tomcat 9\logs\catalina.log' -Destination . -FromSession $s

Remove-PSSession $s
```

## Nota sobre WinRM

Si se pierde la conexión, WinRM permite volver luego sin problema. Con SSH, no.

Esto funciona porque la sesión vive **en el servidor**, no en la conexión:

```powershell
# lanzar algo largo y desconectarse
$s = New-PSSession -ComputerName web01 -Credential $c
Invoke-Command -Session $s -ScriptBlock { Install-WindowsFeature Web-Server } -AsJob
Disconnect-PSSession $s

# más tarde (incluso desde otro equipo, con el mismo usuario)
Get-PSSession -ComputerName web01 -Credential $c
$s = Connect-PSSession -ComputerName web01 -Credential $c -Name $s.Name
Receive-PSSession $s
```

> Con SSH el equivalente es trabajar dentro de `tmux` o `screen` en el servidor.

## Autenticación

| Método | Cuándo | Notas |
|---|---|---|
| **Kerberos** | equipos en dominio, por **nombre** | por defecto; el más seguro |
| **NTLM / Negotiate** | por IP o fuera de dominio | requiere `TrustedHosts` o HTTPS |
| **CredSSP** | necesitas el *double hop* | delega tus credenciales al servidor: **riesgo** si está comprometido |
| **Basic** | cuentas locales, Ansible | solo sobre **HTTPS** |
| **Certificado** | automatización sin contraseñas | mapeo de certificado a cuenta local |

### TrustedHosts (fuera de dominio o por IP)

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts -Value '192.168.1.50' -Concatenate -Force
Get-Item WSMan:\localhost\Client\TrustedHosts
```

> `TrustedHosts = *` "funciona", pero desactiva la comprobación de identidad del servidor
> para cualquier destino: te pueden suplantar el servidor y capturar credenciales NTLM.

## Listener HTTPS

```powershell
# en el servidor: certificado (ideal: de la CA interna; aquí autofirmado para el lab)
$cert = New-SelfSignedCertificate -DnsName 'srv01.lab.local' -CertStoreLocation Cert:\LocalMachine\My

New-Item -Path WSMan:\localhost\Listener -Transport HTTPS -Address * `
    -CertificateThumbPrint $cert.Thumbprint -Force

New-NetFirewallRule -DisplayName 'WinRM HTTPS' -Direction Inbound -Protocol TCP `
    -LocalPort 5986 -Action Allow
```

```powershell
# desde el cliente
Enter-PSSession -ComputerName srv01.lab.local -UseSSL -Credential $c

# con certificado autofirmado (solo lab)
$opt = New-PSSessionOption -SkipCACheck -SkipCNCheck
Enter-PSSession -ComputerName srv01.lab.local -UseSSL -Credential $c -SessionOption $opt
```

## El problema del double hop

```
   TU PC ──(credenciales)──► SRV01 ──(¿credenciales?)──► \\FILESRV\share
                               │
                               └── SRV01 no tiene tu contraseña, solo un ticket
                                   válido para él: el segundo salto da "Access denied"
```

Soluciones, de mejor a peor:

1. **No saltar**: copia primero a tu equipo o ejecuta la acción desde donde están los datos.
2. **Pasar la credencial explícita** al segundo recurso dentro del scriptblock
   (`New-PSDrive -Credential $using:c`).
3. **Kerberos constrained delegation** basada en recursos, configurada en AD.
4. **CredSSP**: funciona siempre, pero expone tus credenciales en SRV01.

## PowerShell sobre SSH (PowerShell 7)

Alternativa multiplataforma a WinRM: funciona entre Windows, Linux y macOS.

```powershell
Enter-PSSession -HostName srv01 -UserName devops          # -HostName = SSH, -ComputerName = WinRM
Invoke-Command -HostName linux01 -UserName devops -ScriptBlock { uname -a }
```

Requiere OpenSSH en el servidor y el subsistema de PowerShell en `sshd_config`:

```
Subsystem powershell c:/progra~1/powershell/7/pwsh.exe -sshs -nologo
```

## Dentro del servidor

Listar features de IIS/Web:

```powershell
Get-WindowsFeature -Name Web-* | Select-Object Name, DisplayName
```

```powershell
Get-WindowsFeature | Where-Object Installed          # qué roles/features hay
Install-WindowsFeature Web-Server -IncludeManagementTools
Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 5
Get-CimInstance Win32_OperatingSystem | Select-Object Caption, LastBootUpTime
```

## Relación con Ansible

Ansible usa exactamente este canal. Variables típicas del inventario (ver
[Ansible en Linux vs Windows](../ansible/02-linux-vs-windows.md)):

```yaml
ansible_connection: winrm
ansible_port: 5986
ansible_winrm_transport: ntlm          # kerberos en dominio
ansible_winrm_server_cert_validation: validate
```

## Atajos útiles en consola

| Atajo | Qué hace |
|---|---|
| `Tab` | autocompleta escribiendo el comienzo del comando |
| `↑` | recupera el último comando |
| `Ctrl + R` | buscar en el histórico por nombre |
| `Ctrl + Espacio` | muestra todas las opciones de autocompletado (PSReadLine) |
| `F7` | historial en ventana (consola clásica) |

## Errores típicos

| Error | Causa habitual |
|---|---|
| `WinRM cannot complete the operation` | servicio WinRM parado, firewall o nombre mal |
| `The WinRM client cannot process the request… TrustedHosts` | conectas por IP o fuera de dominio sin HTTPS: añade a `TrustedHosts` |
| `Access is denied` | el usuario no es administrador ni está en **Remote Management Users** |
| `Access is denied` con cuenta **local** administradora | UAC remoto filtra el token de administradores locales (salvo la cuenta `Administrator` integrada): `LocalAccountTokenFilterPolicy` |
| `Access denied` al tocar un recurso de red dentro de la sesión | double hop |
| `The server certificate on the destination computer has the following errors` | HTTPS con certificado no confiable o CN distinto del nombre usado |
| La sesión se corta en tareas largas | límites de `MaxMemoryPerShellMB` o timeouts: usar `-AsJob` o sesiones desconectadas |
