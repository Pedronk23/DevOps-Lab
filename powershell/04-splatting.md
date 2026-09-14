# PowerShell — Splatting

## Qué es

Técnica que permite pasar parámetros a un cmdlet **desde una variable** (hashtable o array)
en lugar de escribirlos uno a uno en la misma línea.

Al llamar al comando se usa `@variable` en vez de `$variable`.

**Ventaja principal: legibilidad y reutilización.** En vez de una línea kilométrica de
parámetros, los defines antes y los reutilizas cuantas veces quieras.

## Antes y después

```powershell
# sin splatting: línea kilométrica o acentos graves (`) al final de cada línea
Send-MailMessage -From 'alertas@lab.local' -To 'devops@lab.local' -Subject 'Disco lleno' `
    -Body 'Queda <10% en D:' -SmtpServer 'smtp.lab.local' -Port 587 -UseSsl
```

```powershell
# con splatting
$correo = @{
    From       = 'alertas@lab.local'
    To         = 'devops@lab.local'
    Subject    = 'Disco lleno'
    Body       = 'Queda <10% en D:'
    SmtpServer = 'smtp.lab.local'
    Port       = 587
    UseSsl     = $true
}
Send-MailMessage @correo
```

> El acento grave (`` ` ``) como continuación de línea es frágil: **un espacio invisible
> detrás** rompe el comando. El splatting evita ese problema.

(`Send-MailMessage` está marcado como obsoleto por Microsoft, pero sirve bien para ilustrar un
cmdlet con muchos parámetros.)

## Splatting con hashtable — `@{}` (con llaves)

Para parámetros con nombre:

```powershell
$params = @{
    Name              = "Juan García"
    SamAccountName    = "juan.garcia"
    Department        = "IT"
    Enabled           = $true
    AccountPassword   = (ConvertTo-SecureString "Pass123!" -AsPlainText -Force)
}

New-ADUser @params
```

> ⚠️ El ejemplo lleva la contraseña **en claro** en el script: vale para entender la
> sintaxis, no para un script real. Mejor pedirla o sacarla de un gestor de secretos:
> `AccountPassword = (Read-Host -AsSecureString "Contraseña inicial")`.

- Las **claves** son los nombres de los parámetros, sin guion.
- Los **switches** (parámetros sin valor, como `-Recurse` o `-Force`) se ponen a `$true`.

## Splatting con array — `@()` (con paréntesis)

Para parámetros **posicionales**, es decir, sin nombre. Los valores se asignan en orden a
los parámetros posicionales del cmdlet.

```powershell
$params = @("origen.txt", "destino.txt")
Copy-Item @params

# Equivale a:
Copy-Item "origen.txt" "destino.txt"
#              ^Path        ^Destination
```

> El array depende del **orden** de los parámetros: se rompe si el cmdlet cambia. En la
> práctica casi siempre se usa la hashtable.

## Combinar splatting con parámetros sueltos

```powershell
$comun = @{
    ComputerName = 'srv01', 'srv02'
    Credential   = $c
}

Invoke-Command @comun -ScriptBlock { Get-Service W3SVC }
Invoke-Command @comun -ScriptBlock { Restart-Service W3SVC }
```

Se pueden usar **varios** splats en la misma llamada: `Get-ChildItem @origen @filtros`.

## Parámetros condicionales

Aquí el splatting es insustituible: añadir parámetros solo cuando hacen falta.

```powershell
$params = @{
    Path    = 'D:\logs'
    Filter  = '*.log'
}

if ($Recursivo) {
    $params['Recurse'] = $true
}
if ($Credencial) {
    $params.Credential = $Credencial
}

Get-ChildItem @params
```

Sin splatting habría que escribir cuatro versiones del mismo comando con `if/else`.

## Reutilizar y extender una base

```powershell
$base = @{
    Uri         = 'https://api.lab.local/v1/despliegues'
    Headers     = @{ Authorization = "Bearer $token" }
    ContentType = 'application/json'
}

$lista = Invoke-RestMethod @base -Method Get

$nuevo = $base + @{
    Method = 'Post'
    Body   = (@{ app = 'web'; version = '1.4.2' } | ConvertTo-Json)
}
Invoke-RestMethod @nuevo
```

> Sumar hashtables con `+` **falla si hay claves repetidas**. Para sobrescribir, clona y
> asigna: `$nuevo = $base.Clone(); $nuevo.Method = 'Post'`.

## Reenviar parámetros en funciones: `@PSBoundParameters`

`$PSBoundParameters` es una hashtable automática con los parámetros que recibió la función.
Permite crear "envoltorios" que pasan todo al cmdlet interno:

```powershell
function Get-LogGrande {
    param(
        [string]$Path,
        [switch]$Recurse,
        [int]$MinMB = 100
    )

    $null = $PSBoundParameters.Remove('MinMB')      # este no lo entiende Get-ChildItem

    Get-ChildItem @PSBoundParameters -File |
        Where-Object Length -gt ($MinMB * 1MB)
}

Get-LogGrande -Path D:\logs -Recurse -MinMB 500
```

Y `@args` reenvía los argumentos sin declarar: `function k { kubectl @args }`.

## Hashtable ordenada

Una hashtable normal **no garantiza el orden** de las claves. Para splatting da igual, pero
si además la muestras o la conviertes a JSON, usa `[ordered]`:

```powershell
$params = [ordered]@{
    Name  = 'web'
    Port  = 8080
    State = 'Started'
}
```

## Errores típicos

| Síntoma | Causa |
|---|---|
| `Cannot convert 'System.Collections.Hashtable' to the type…` | escribiste `$params` en vez de `@params` |
| `Cannot find path '…\System.Collections.Hashtable'` | lo mismo: el hashtable entero se pasó como `-Path` |
| `A parameter cannot be found that matches parameter name 'X'` | una clave del hashtable no existe como parámetro de ese cmdlet (erratas: `Computername` vale, `Computer` no) |
| `Parameter set cannot be resolved` | combinaste parámetros de conjuntos incompatibles |
| `Item has already been added. Key in dictionary: 'X'` | sumaste dos hashtables con la misma clave |
| El splatting "no funciona" con un método .NET | solo funciona con **comandos** (cmdlets, funciones, scripts), no con `$objeto.Metodo()` |
