# PowerShell — `Get-PSDrive` y proveedores

`Get-PSDrive` lista las unidades disponibles en tu sesión actual de PowerShell: tanto las
**reales** (disco duro) como las **virtuales** (registro, variables, etc.).

## La idea: todo se navega como un sistema de ficheros

Un **proveedor** (*PSProvider*) es un adaptador que presenta un almacén de datos como si
fuera un árbol de carpetas. Por eso **los mismos cmdlets** sirven para todo:

```
                    Get-ChildItem / Get-Item / New-Item / Remove-Item / Set-ItemProperty
                                              │
        ┌──────────────┬──────────────┬───────┴──────┬──────────────┬──────────────┐
        ▼              ▼              ▼              ▼              ▼              ▼
   FileSystem      Registry      Certificate     Environment     Alias         WSMan
   C:\  D:\        HKLM: HKCU:   Cert:           Env:            Alias:        WSMan:
```

```powershell
Get-PSProvider        # proveedores cargados y qué unidades expone cada uno
```

## File System (las únicas "reales" en disco)

| Unidad | Qué es |
|---|---|
| `C:` | disco duro principal |
| `Temp:` | acceso directo a la carpeta temporal (apunta dentro de `C:\`) |

## Proveedores virtuales (no ocupan espacio en disco)

| Proveedor | Contenido |
|---|---|
| `Alias:` | atajos de comando. Ej.: `ls` es alias de `Get-ChildItem`. Se ven con `Get-Alias` |
| `Env:` | variables de entorno del sistema: `PATH`, `USERNAME`, … Se ven con `ls Env:` |
| `Function:` | funciones cargadas en tu sesión |
| `Variable:` | variables definidas en tu sesión: `$null`, `$true`, `$false`, … |

## Registry (el registro de Windows)

| Unidad | Significado | Contenido |
|---|---|---|
| `HKCU:` | `HKEY_CURRENT_USER` | configuración del usuario actual |
| `HKLM:` | `HKEY_LOCAL_MACHINE` | configuración global del sistema y programas instalados |

## Otros

| Unidad | Contenido |
|---|---|
| `Cert:` | almacén de certificados digitales de Windows (SSL, firma de código, …) |
| `WSMan:` | configuración de WS-Management, el protocolo que usa PowerShell Remoting para conectarse a otros equipos |

```powershell
Get-PSDrive
ls Env:
ls HKLM:\SOFTWARE
```

## Env: variables de entorno

```powershell
$env:COMPUTERNAME
$env:PATH -split ';'                         # PATH legible, una ruta por línea
$env:MI_VARIABLE = 'valor'                   # solo para esta sesión (y procesos hijos)
Remove-Item Env:\MI_VARIABLE

# persistente (para nuevas sesiones): User o Machine (Machine requiere administrador)
[Environment]::SetEnvironmentVariable('JAVA_HOME', 'C:\apps\MiAppWeb\Java', 'Machine')
[Environment]::GetEnvironmentVariable('JAVA_HOME', 'Machine')
```

> Tras cambiar una variable persistente, **las consolas y servicios ya abiertos no la ven**:
> hay que abrir una consola nueva o reiniciar el servicio.

## Registro: leer y escribir

En el registro, las **claves** son las "carpetas" y los **valores** son las "propiedades".

```
   HKLM:\SOFTWARE\MiApp            ← clave   (Get-ChildItem, New-Item)
        ├── Puerto   = 8080        ← valor   (Get-ItemProperty, New-ItemProperty)
        └── Entorno  = "prod"
```

```powershell
# versión de Windows
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' |
    Select-Object ProductName, DisplayVersion, CurrentBuild

# leer un solo valor
Get-ItemPropertyValue 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server' -Name fDenyTSConnections

# crear clave y valores
New-Item -Path HKLM:\SOFTWARE\MiApp -Force
New-ItemProperty -Path HKLM:\SOFTWARE\MiApp -Name Puerto -Value 8080 -PropertyType DWord
Set-ItemProperty -Path HKLM:\SOFTWARE\MiApp -Name Entorno -Value 'prod'
Remove-ItemProperty -Path HKLM:\SOFTWARE\MiApp -Name Entorno
```

> Curiosidad: en Windows 11, `ProductName` sigue diciendo "Windows 10". Para distinguirlo,
> mira `CurrentBuild` (22000 o superior = Windows 11).

| `-PropertyType` | Tipo en regedit | Uso |
|---|---|---|
| `String` | `REG_SZ` | texto |
| `ExpandString` | `REG_EXPAND_SZ` | texto con `%VARIABLES%` |
| `DWord` | `REG_DWORD` | entero de 32 bits (flags 0/1) |
| `QWord` | `REG_QWORD` | entero de 64 bits |
| `MultiString` | `REG_MULTI_SZ` | lista de textos |
| `Binary` | `REG_BINARY` | bytes |

### Programas instalados

```powershell
$rutas = 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*',
         'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*'   # 32 bits

Get-ItemProperty $rutas |
    Where-Object DisplayName |
    Select-Object DisplayName, DisplayVersion, Publisher |
    Sort-Object DisplayName
```

> Evita `Get-WmiObject Win32_Product` para esto: es lentísimo y **lanza una reparación**
> de cada paquete MSI al consultarlo.

### Otras ramas del registro

`HKCR` y `HKU` no vienen montadas. Se crean con `New-PSDrive`:

```powershell
New-PSDrive -Name HKU -PSProvider Registry -Root HKEY_USERS
Get-ChildItem HKU:\
```

> Antes de tocar el registro en un servidor: `reg export HKLM\SOFTWARE\MiApp copia.reg`.

## Cert: almacén de certificados

```
   Cert:\
   ├── LocalMachine\        (del equipo: IIS, servicios, WinRM HTTPS)
   │   ├── My               ← "Personal": certificados con clave privada
   │   ├── Root             ← CAs raíz de confianza
   │   └── CA               ← CAs intermedias
   └── CurrentUser\         (del usuario)
       └── My
```

```powershell
# certificados del equipo con fecha de caducidad y huella
Get-ChildItem Cert:\LocalMachine\My | Select-Object Subject, NotAfter, Thumbprint

# los que caducan en los próximos 30 días
Get-ChildItem Cert:\LocalMachine\My -Recurse |
    Where-Object { $_.NotAfter -lt (Get-Date).AddDays(30) } |
    Select-Object Subject, NotAfter

# importar la CA interna como raíz de confianza
Import-Certificate -FilePath .\ca-lab.crt -CertStoreLocation Cert:\LocalMachine\Root

# importar un certificado con su clave privada
$pass = Read-Host -AsSecureString "Contraseña del PFX"
Import-PfxCertificate -FilePath .\web.pfx -CertStoreLocation Cert:\LocalMachine\My -Password $pass
```

Consolas gráficas equivalentes: `certlm.msc` (equipo) y `certmgr.msc` (usuario). Más contexto
en [CA y certificados](../seguridad/08-ca-y-certificados.md).

## WSMan: configuración de remoting

```powershell
Get-ChildItem WSMan:\localhost\Listener                     # listeners HTTP/HTTPS
Get-Item WSMan:\localhost\Client\TrustedHosts                # equipos de confianza (sin Kerberos)
Set-Item WSMan:\localhost\Client\TrustedHosts -Value 'srv01.lab.local' -Concatenate -Force
Get-Item WSMan:\localhost\Shell\MaxMemoryPerShellMB
```

Ver [WinRM y remoting](05-winrm-y-remoting.md).

## Crear unidades propias

```powershell
# atajo a una carpeta de trabajo (solo para esta sesión)
New-PSDrive -Name repo -PSProvider FileSystem -Root C:\Users\pedro\git
Set-Location repo:

# mapear un recurso compartido con otras credenciales, visible en el Explorador
$c = Get-Credential
New-PSDrive -Name S -PSProvider FileSystem -Root \\srv01\despliegues -Credential $c -Persist

Remove-PSDrive -Name repo
```

| Opción | Efecto |
|---|---|
| sin `-Persist` | solo existe dentro de PowerShell, en esta sesión |
| `-Persist` | unidad de red real (como `net use`); el nombre debe ser **una letra** |
| `-Scope Global` | visible fuera del script que la crea |
