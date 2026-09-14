# PowerShell — Alias y equivalencias

Los alias son atajos de comando. Se listan con `Get-Alias`.

```powershell
Get-Alias                              # todos
Get-Alias ls                           # ¿a qué apunta ls?
Get-Alias -Definition Get-ChildItem    # ¿qué alias tiene este cmdlet?
Get-Command ls                         # alias, función, cmdlet o ejecutable
```

## Ficheros y navegación

| Alias | Comando real | Equivalente |
|---|---|---|
| `ls` | `Get-ChildItem` | Linux: `ls` |
| `dir` | `Get-ChildItem` | CMD: `dir` |
| `gci` | `Get-ChildItem` | alias corto |
| `cd` | `Set-Location` | `cd` |
| `pwd` | `Get-Location` | Linux: `pwd` |
| `cat` | `Get-Content` | Linux: `cat` |
| `type` | `Get-Content` | CMD: `type` |
| `gc` | `Get-Content` | alias corto |
| `sc` | `Set-Content` | alias corto |
| `gi` | `Get-Item` | alias corto |
| `ni` | `New-Item` | alias corto |
| `ri` | `Remove-Item` | alias corto |
| `mi` | `Move-Item` | alias corto |
| `cp` | `Copy-Item` | Linux: `cp` |
| `mv` | `Move-Item` | Linux: `mv` |
| `rm` | `Remove-Item` | Linux: `rm` |
| `del` | `Remove-Item` | CMD: `del` |
| `mkdir` | `New-Item -ItemType Directory` | Linux: `mkdir` |
| `md` | `mkdir` | CMD: `md` |
| `rmdir` | `Remove-Item` | Linux: `rmdir` |
| `ren` | `Rename-Item` | CMD: `ren` |
| `gp` | `Get-ItemProperty` | registro / config |

> Los alias **no aceptan las opciones del comando original**. `ls -la` no funciona en
> PowerShell: `ls` es `Get-ChildItem` y lo que existe es `Get-ChildItem -Force` (ocultos).
> Igual con `rm -rf` → `Remove-Item -Recurse -Force`.

## Salida, sistema y red

| Alias | Comando real | Equivalente |
|---|---|---|
| `echo` | `Write-Output` | `echo` |
| `cls` | `Clear-Host` | `clear` |
| `clear` | `Clear-Host` | Linux: `clear` |
| `man` | `help` | Linux: `man` |
| `ps` | `Get-Process` | Linux: `ps` |
| `kill` | `Stop-Process` | Linux: `kill` |
| `sleep` | `Start-Sleep` | Linux: `sleep` |
| `sort` | `Sort-Object` | Linux: `sort` |
| `curl` | `Invoke-WebRequest` | Linux: `curl` |
| `wget` | `Invoke-WebRequest` | Linux: `wget` |
| `history` | `Get-History` | bash `history` |
| `h` | `Get-History` | atajo |
| `iex` | `Invoke-Expression` | ejecutar un string |

## Pipeline y filtrado

| Alias | Comando real | Para qué |
|---|---|---|
| `where` | `Where-Object` | filtrar |
| `?` | `Where-Object` | filtro rápido |
| `%` | `ForEach-Object` | iterar en pipeline |
| `select` | `Select-Object` | elegir propiedades o los N primeros |
| `ft` / `fl` | `Format-Table` / `Format-List` | presentación en tabla / lista |
| `gm` | `Get-Member` | ver propiedades y métodos de un objeto |
| `\|` | (pipe) | "toma esto y dáselo al siguiente" |

## Trampas con los alias

| Trampa | Detalle |
|---|---|
| `curl` / `wget` | solo son alias en **Windows PowerShell 5.1**. En PowerShell 7 se quitaron y llaman al `curl.exe` real |
| `sc` | en 5.1 es `Set-Content`, no el gestor de servicios: escribe **`sc.exe`** (`sc.exe query W3SVC`). En PowerShell 7 ese alias ya no existe |
| `where` | en PowerShell es `Where-Object`; el `where` de CMD (el "which" de Windows) es **`where.exe`** |
| PowerShell 7 en Linux/macOS | no define los alias que chocan con binarios nativos (`ls`, `cp`, `mv`, `rm`, `cat`, `ps`…): ejecuta los del sistema |
| Scripts | los alias hacen el código **ilegible y no portable**: en scripts, siempre el nombre completo |

> PSScriptAnalyzer lo marca con la regla `PSAvoidUsingCmdletAliases`. VS Code con la extensión
> de PowerShell ofrece "expandir alias" automáticamente.

## Crear alias propios

```powershell
Set-Alias -Name k -Value kubectl
Set-Alias -Name np -Value notepad.exe
```

Un alias **no puede llevar parámetros**. Para eso, una función:

```powershell
function kgp { kubectl get pods @args }       # kgp -n monitoring
function .. { Set-Location .. }
```

Para que duren entre sesiones, al **perfil**:

```powershell
$PROFILE                                       # ruta del perfil del usuario actual
if (-not (Test-Path $PROFILE)) { New-Item -ItemType File -Path $PROFILE -Force }
notepad $PROFILE
. $PROFILE                                     # recargar sin abrir otra consola
```

## Windows PowerShell vs PowerShell 7

| | Windows PowerShell 5.1 | PowerShell 7 |
|---|---|---|
| Ejecutable | `powershell.exe` | `pwsh.exe` |
| Base | .NET Framework | .NET (moderno) |
| Plataformas | solo Windows | Windows, Linux, macOS |
| Viene instalado | sí | no (winget, MSI) |
| `&&` y `\|\|` | no | sí |
| Codificación por defecto | varía (UTF-16 con `>`) | UTF-8 sin BOM |
| Módulos antiguos (algunos de AD, WSUS…) | todos | la mayoría (con capa de compatibilidad) |

```powershell
$PSVersionTable.PSVersion
```

## Equivalencias de tareas (no son alias)

| Tarea | Linux | CMD | PowerShell |
|---|---|---|---|
| Buscar texto | `grep error app.log` | `findstr error app.log` | `Select-String error app.log` |
| Primeras líneas | `head -20 f` | — | `Get-Content f -TotalCount 20` |
| Seguir un log | `tail -f app.log` | — | `Get-Content app.log -Wait -Tail 50` |
| Contar líneas | `wc -l f` | `find /c /v "" f` | `(Get-Content f).Count` |
| ¿Dónde está un binario? | `which kubectl` | `where kubectl` | `Get-Command kubectl` |
| Crear fichero vacío | `touch f` | `type nul > f` | `New-Item f` |
| Buscar ficheros | `find . -name "*.log"` | `dir /s *.log` | `Get-ChildItem -Recurse -Filter *.log` |
| Variables de entorno | `env` | `set` | `Get-ChildItem Env:` |
| Definir variable | `export VAR=x` | `set VAR=x` | `$env:VAR = 'x'` |
| IPs | `ip a` | `ipconfig` | `Get-NetIPAddress` |
| Puertos en escucha | `ss -tlnp` | `netstat -ano` | `Get-NetTCPConnection -State Listen` |
| Ping | `ping host` | `ping host` | `Test-Connection host` |
| ¿Puerto abierto? | `nc -zv host 443` | — | `Test-NetConnection host -Port 443` |
| Resolver DNS | `dig host` | `nslookup host` | `Resolve-DnsName host` |
| Espacio en disco | `df -h` | — | `Get-Volume` / `Get-PSDrive -PSProvider FileSystem` |
| Servicios | `systemctl status x` | `sc query x` | `Get-Service x` |
| Reiniciar servicio | `systemctl restart x` | `net stop x && net start x` | `Restart-Service x` |
| Logs del sistema | `journalctl -u x` | — | `Get-WinEvent -LogName System -MaxEvents 50` |
| Permisos | `chmod` / `chown` | `icacls` | `Get-Acl` / `Set-Acl` (o `icacls`) |
| Comprimir | `tar czf` / `zip` | — | `Compress-Archive` / `Expand-Archive` |
| Hash de fichero | `sha256sum f` | `certutil -hashfile f SHA256` | `Get-FileHash f` |
