# PowerShell — Navegación, pilas y pipeline

## Navegación básica

- `cd` → `Set-Location`: cambia de directorio.
- `ls`, `dir`, `gci` → `Get-ChildItem`: muestra el contenido de una carpeta.
  `dir` viene de MS-DOS, `ls` de Linux; en PowerShell son el mismo cmdlet.
- `mkdir` → alias de `New-Item -ItemType Directory`.

| Comando | Qué hace |
|---|---|
| `Set-Location -` / `cd -` | vuelve al directorio anterior (PowerShell 6.2+) |
| `Set-Location ~` | al perfil del usuario |
| `Get-ChildItem -Force` | incluye ocultos y de sistema |
| `Get-ChildItem -Recurse -Filter *.log` | búsqueda recursiva |
| `Get-ChildItem -Directory` / `-File` | solo carpetas / solo ficheros |
| `Resolve-Path .\conf\*.xml` | rutas completas de lo que coincide |
| `Invoke-Item .` / `ii .` | abre la carpeta actual en el Explorador |

## Pila de ubicaciones (LIFO)

PowerShell mantiene internamente una **pila** de ubicaciones. Funciona en **LIFO**
(*Last In, First Out*).

- `Push-Location` (`pushd`): te mueve a una nueva ubicación y guarda en memoria dónde estabas.
  Se pueden apilar varios pushes seguidos.
- `Pop-Location` (`popd`): saca el último elemento de la pila y te lleva allí.
  No hace falta indicarle ninguna ruta. Cada `pop` te devuelve un paso atrás,
  en orden inverso al apilado.

```powershell
pushd C:\inetpub\wwwroot
pushd C:\Windows\System32
popd   # vuelve a C:\inetpub\wwwroot
popd   # vuelve al punto inicial
```

```
   inicio: C:\Users\pedro
   pushd C:\inetpub\wwwroot    pila: [C:\Users\pedro]
   pushd C:\Windows\System32   pila: [C:\Users\pedro, C:\inetpub\wwwroot]
   popd  → C:\inetpub\wwwroot  pila: [C:\Users\pedro]
   popd  → C:\Users\pedro      pila: []
```

```powershell
Get-Location -Stack                         # ver la pila
Push-Location C:\temp -StackName despliegue # pilas con nombre, independientes
Pop-Location -StackName despliegue
```

Uso típico en scripts: entrar en una carpeta y **volver pase lo que pase**:

```powershell
Push-Location C:\apps\MiAppWeb
try {
    .\instalar.ps1
}
finally {
    Pop-Location
}
```

## La idea clave: el pipeline pasa OBJETOS, no texto

```
   BASH                                     POWERSHELL
   ────                                     ──────────
   ps aux | grep java | awk '{print $2}'    Get-Process java | Select-Object Id
     │         │            │                  │                   │
    texto ─► texto ────► recortar columna    objetos ──────────► propiedad
                         (frágil: depende    (Process con .Id,   Id por nombre
                          del formato)        .CPU, .Path…)
```

Consecuencia: no hay que "parsear" columnas. Se filtra y se ordena por **propiedades**.

```powershell
Get-Service | Get-Member                   # qué propiedades y métodos tiene cada objeto
Get-Service W3SVC | Select-Object *        # todas las propiedades con su valor
```

## Pipeline y filtrado

```powershell
gci | ? -Property Name -Match web
```

Desglose:

| Parte | Qué hace |
|---|---|
| `gci` | genera la lista de archivos y carpetas |
| `\|` (pipe) | coge todo lo que produjo `gci` y lo pasa como entrada al comando siguiente |
| `?` | alias de `Where-Object`: filtra la lista que le llega, quedándose solo con lo que cumple la condición |
| `-Property Name` | parámetro `-Property` con valor `Name`: mira la propiedad "nombre" de cada elemento |
| `-Match web` | parámetro `-Match` con valor `web`: se queda con los que contengan "web" en el nombre |

Otro ejemplo, servicios cuyo nombre contenga "web" (`*` = cualquier cosa):

```powershell
Get-Service -Name "*web*"
```

### Dos sintaxis de `Where-Object`

```powershell
# simplificada: una sola condición
Get-Service | Where-Object Status -eq 'Running'

# con scriptblock: varias condiciones; $_ es el objeto actual
Get-Service | Where-Object { $_.Status -eq 'Stopped' -and $_.StartType -eq 'Automatic' }
```

> Filtra **lo antes posible**. `Get-ChildItem -Filter *.log` es mucho más rápido que
> `Get-ChildItem | Where-Object Name -like *.log`, porque el filtro lo aplica el proveedor.

## Cmdlets del pipeline

| Cmdlet | Para qué | Ejemplo |
|---|---|---|
| `Where-Object` | filtrar | `? CPU -gt 100` |
| `Select-Object` | elegir propiedades, primeros/últimos N | `select Name, Id -First 5` |
| `Sort-Object` | ordenar | `sort CPU -Descending` |
| `ForEach-Object` | hacer algo con cada uno | `% { $_.Name.ToUpper() }` |
| `Group-Object` | agrupar y contar | `group Status` |
| `Measure-Object` | contar, sumar, media | `measure Length -Sum` |
| `Tee-Object` | guardar y seguir | `tee -Variable procesos` |
| `Format-Table` / `Format-List` | presentar | `ft -AutoSize` |
| `Export-Csv` / `ConvertTo-Json` / `Out-File` | sacar a fichero | `Export-Csv inf.csv -NoTypeInformation` |

## Operadores de comparación

| Operador | Significado | Ejemplo |
|---|---|---|
| `-eq` / `-ne` | igual / distinto | `$_.Status -eq 'Running'` |
| `-gt` / `-ge` / `-lt` / `-le` | mayor / mayor o igual / menor / menor o igual | `$_.Length -gt 1GB` |
| `-like` / `-notlike` | comodines `*` `?` | `$_.Name -like 'web*'` |
| `-match` / `-notmatch` | expresión regular (rellena `$Matches`) | `$_.Name -match '^srv\d+$'` |
| `-contains` / `-notcontains` | ¿la **colección** contiene el valor? | `@('a','b') -contains 'a'` |
| `-in` / `-notin` | ¿el valor está **en** la colección? | `'a' -in @('a','b')` |
| `-and` / `-or` / `-not` (`!`) | lógicos | |

> No distinguen mayúsculas por defecto. Para que sí: `-ceq`, `-clike`, `-cmatch`.
> `=` es **asignación**, nunca comparación: `if ($x = 5)` siempre es verdadero.

## Recetas

```powershell
# los 5 procesos que más CPU han consumido
Get-Process | Sort-Object CPU -Descending | Select-Object -First 5 Name, Id, CPU

# servicios automáticos que están parados (candidatos a revisar tras un reinicio)
Get-Service | Where-Object { $_.StartType -eq 'Automatic' -and $_.Status -ne 'Running' }

# ficheros de log de más de 100 MB, con tamaño legible
Get-ChildItem D:\logs -Recurse -File |
    Where-Object Length -gt 100MB |
    Sort-Object Length -Descending |
    Select-Object FullName, @{Name = 'MB'; Expression = { [math]::Round($_.Length / 1MB, 1) }}

# cuánto ocupa una carpeta
(Get-ChildItem C:\apps -Recurse -File | Measure-Object Length -Sum).Sum / 1GB

# recuento de servicios por estado
Get-Service | Group-Object Status | Select-Object Name, Count

# errores del registro de eventos de las últimas 24 h
Get-WinEvent -FilterHashtable @{ LogName = 'Application'; Level = 2; StartTime = (Get-Date).AddDays(-1) }

# exportar un inventario a CSV (se abre en Excel)
Get-Service | Select-Object Name, DisplayName, Status, StartType |
    Export-Csv servicios.csv -NoTypeInformation -Encoding utf8
```

> `@{Name = …; Expression = { … }}` es una **propiedad calculada**: añade columnas que no
> existen en el objeto original.

## Formatear siempre al final

```
   Get-Service | Where-Object … | Sort-Object … | Format-Table     ✔
   Get-Service | Format-Table | Export-Csv servicios.csv            ✘
                     └── a partir de aquí ya no hay objetos Service,
                         solo "instrucciones de formato": el CSV sale basura
```

`Format-*` es para **mirar** en pantalla. Para ficheros o para seguir procesando, usa
`Select-Object`.

## Evitar sustos: `-WhatIf` y `-Confirm`

```powershell
Get-ChildItem C:\temp -Recurse -Filter *.tmp | Remove-Item -WhatIf   # dice qué haría, sin hacerlo
Stop-Service W3SVC -Confirm                                          # pregunta antes
```

Antes de cualquier `Remove-Item` en un pipeline, primero `-WhatIf`.
