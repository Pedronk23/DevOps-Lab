# Python — pandas

Librería de Python (`import pandas as pd`) especializada en **manipulación y análisis de
datos**: análisis de datos, ciencia de datos, automatización, ETL, tratamiento de CSV/Excel…

## Instalación

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\Activate.ps1
pip install pandas openpyxl        # openpyxl hace falta para leer/escribir .xlsx
```

## Operaciones básicas

```python
import pandas as pd

# Leer archivos: CSV / Excel / JSON
df = pd.read_csv("datos.csv")

# Filtrar datos
mayores = df[df["edad"] > 26]

# Seleccionar columnas
df[["nombre", "edad"]]

# Modificar datos
df["edad"] = df["edad"] + 1

# Agrupar datos
df.groupby("departamento")["salario"].mean()

# Exportar resultados
df.to_csv("resultado.csv")
```

## Notas

- `df` = DataFrame: tabla bidimensional con índice y columnas con nombre.
- Los filtros devuelven una vista/copia nueva; no modifican el original salvo asignación.

## Series y DataFrame

```
                 columnas ─────────────────────────────►
                 hostname     entorno   cpu    ram_gb
   índice  0     web01        prod      82.5   16
     │     1     web02        prod      41.0   16
     │     2     db01         prod      67.3   64
     ▼     3     web-dev01    dev       12.8   8
                 └────┬────┘
                 cada columna es una SERIES (una lista con índice y un tipo)
```

| Objeto | Qué es |
|---|---|
| `Series` | una columna: valores + índice + tipo (`dtype`) |
| `DataFrame` | varias Series que comparten índice |
| `Index` | las etiquetas de las filas (por defecto 0, 1, 2…) |

## Leer datos

```python
# CSV exportado desde un Excel en español: separador ";" y decimales con ","
df = pd.read_csv(
    "inventario.csv",
    sep=";",
    decimal=",",
    encoding="utf-8-sig",           # quita el BOM si lo trae (ver fundamentos/parsers-y-codificacion)
    parse_dates=["fecha_alta"],
    dtype={"codigo_postal": str},   # que no convierta "08001" en 8001
    usecols=["hostname", "entorno", "cpu", "ram_gb", "fecha_alta", "codigo_postal"],
)

df = pd.read_excel("inventario.xlsx", sheet_name="Servidores")
df = pd.read_json("salida.json")
```

### JSON anidado (APIs, kubectl, AWX)

```python
import json
import subprocess

import pandas as pd

salida = subprocess.run(
    ["kubectl", "get", "pods", "-A", "-o", "json"],
    capture_output=True, text=True, check=True,
)
pods = json.loads(salida.stdout)["items"]

df = pd.json_normalize(pods)               # aplana: metadata.name, status.phase, …
resumen = (
    df.groupby(["metadata.namespace", "status.phase"])
      .size()
      .unstack(fill_value=0)
)
print(resumen)
```

```
status.phase         Failed  Pending  Running  Succeeded
metadata.namespace
kube-system               0        0       14          0
monitoring                0        1        9          0
```

## Explorar

| Método | Qué muestra |
|---|---|
| `df.head(10)` / `df.tail()` | primeras / últimas filas |
| `df.shape` | `(filas, columnas)` |
| `df.info()` | columnas, tipos y valores no nulos |
| `df.describe()` | estadísticas de las columnas numéricas |
| `df.dtypes` | tipo de cada columna |
| `df.columns` | nombres de columnas |
| `df["entorno"].value_counts()` | recuento por valor |
| `df["entorno"].unique()` | valores distintos |
| `df.isna().sum()` | nulos por columna |

## Seleccionar: `loc` vs `iloc`

| Forma | Selecciona por | Ejemplo |
|---|---|---|
| `df["col"]` | nombre de columna | `df["cpu"]` |
| `df[["a", "b"]]` | varias columnas | `df[["hostname", "cpu"]]` |
| `df.loc[filas, columnas]` | **etiquetas** y condiciones | `df.loc[df["cpu"] > 80, ["hostname", "cpu"]]` |
| `df.iloc[filas, columnas]` | **posición** numérica | `df.iloc[0:5, 0:2]` |

## Filtrar

```python
# varias condiciones: & (y), | (o), ~ (no), y SIEMPRE paréntesis
criticos = df[(df["entorno"] == "prod") & (df["cpu"] > 80)]

windows = df[df["so"].isin(["Windows Server 2019", "Windows Server 2022"])]
webs = df[df["hostname"].str.startswith("web")]
sin_dominio = df[~df["hostname"].str.contains(r"\.lab\.local$", regex=True)]

# lo mismo en texto, a veces más legible
criticos = df.query("entorno == 'prod' and cpu > 80")
```

> Con `and`/`or` de Python en vez de `&`/`|` sale el error
> `The truth value of a Series is ambiguous`.

## Modificar y limpiar

```python
# asignar a un subconjunto: con loc, en una sola operación
df.loc[df["entorno"] == "prod", "prioridad"] = "alta"

# columnas nuevas
df["ram_mb"] = df["ram_gb"] * 1024
df["dias_desde_alta"] = (pd.Timestamp.today() - df["fecha_alta"]).dt.days

# limpieza
df.columns = df.columns.str.strip().str.lower()        # cabeceras con espacios o mayúsculas
df = df.rename(columns={"host name": "hostname"})
df = df.drop_duplicates(subset="hostname")
df["cpu"] = pd.to_numeric(df["cpu"], errors="coerce")  # lo que no sea número → NaN
df["entorno"] = df["entorno"].fillna("desconocido")
df = df.dropna(subset=["hostname"])
df = df.sort_values(["entorno", "cpu"], ascending=[True, False])
```

> **Filtros y copias**: en pandas 3 (*Copy-on-Write*) el resultado de un filtro se comporta
> siempre como una copia: `criticos["x"] = 1` nunca cambia `df`. Para cambiar el original,
> `df.loc[condición, "x"] = 1`. En pandas 1.x/2.x ese mismo código lanza
> `SettingWithCopyWarning` y el resultado es ambiguo; `criticos = df[...].copy()` deja claro
> que quieres una tabla independiente en cualquier versión.

## Agrupar

```python
resumen = df.groupby("entorno").agg(
    servidores=("hostname", "count"),
    cpu_media=("cpu", "mean"),
    ram_total_gb=("ram_gb", "sum"),
).round(1)
```

```
            servidores  cpu_media  ram_total_gb
entorno
dev                 12       18.4            96
prod                31       56.9          1024
```

```python
# tabla dinámica: servidores por entorno y sistema operativo
df.pivot_table(index="entorno", columns="so", values="hostname",
               aggfunc="count", fill_value=0)
```

## Combinar tablas

```python
# ¿qué servidores no tienen responsable asignado?
cruce = servidores.merge(responsables, on="app", how="left", indicator=True)
huerfanos = cruce[cruce["_merge"] == "left_only"]

# apilar ficheros con las mismas columnas
todos = pd.concat([pd.read_csv(f) for f in ["enero.csv", "febrero.csv"]], ignore_index=True)
```

| `how=` | Resultado |
|---|---|
| `inner` | solo las filas que están en las dos tablas |
| `left` | todas las de la izquierda (con NaN si no cruzan) |
| `right` | todas las de la derecha |
| `outer` | todas las de ambas |

## Exportar

```python
df.to_csv("resultado.csv", index=False)                  # sin la columna del índice
df.to_csv("para_excel.csv", index=False, sep=";", encoding="utf-8-sig")   # Excel lo abre bien
df.to_json("resultado.json", orient="records", indent=2, force_ascii=False)

with pd.ExcelWriter("informe.xlsx") as writer:
    criticos.to_excel(writer, sheet_name="CPU alta", index=False)
    resumen.to_excel(writer, sheet_name="Resumen")
```

> Sin `index=False`, al volver a leer el CSV aparece una columna `Unnamed: 0`.

## Receta: analizar un access log de nginx

```python
import pandas as pd

patron = (
    r'(?P<ip>\S+) \S+ \S+ \[(?P<fecha>[^\]]+)\] '
    r'"(?P<metodo>\S+) (?P<ruta>\S+) \S+" (?P<codigo>\d{3}) (?P<bytes>\d+|-)'
)

with open("access.log", encoding="utf-8") as f:
    lineas = pd.Series(f.read().splitlines())

df = lineas.str.extract(patron)
df["fecha"] = pd.to_datetime(df["fecha"], format="%d/%b/%Y:%H:%M:%S %z")
df["codigo"] = df["codigo"].astype(int)

print(df["ip"].value_counts().head(10))                                    # top IPs
print(df[df["codigo"] >= 500].groupby("ruta").size().nlargest(10))          # rutas con más 5xx
print(df.set_index("fecha").resample("5min").size())                        # peticiones cada 5 min
```

## Errores típicos

| Error | Causa habitual |
|---|---|
| `KeyError: 'hostname'` | la cabecera trae espacios, BOM o mayúsculas: `print(df.columns.tolist())` |
| `UnicodeDecodeError: 'utf-8' codec can't decode` | fichero en Windows-1252: `encoding="cp1252"` (o `latin-1`) |
| Todo en una sola columna | separador equivocado: `sep=";"` |
| Números leídos como texto (`str` / `object`) | decimales con coma (`decimal=","`) o celdas con texto: `pd.to_numeric(errors="coerce")` |
| `ValueError: The truth value of a Series is ambiguous` | `and`/`or` en vez de `&`/`\|`, o faltan paréntesis |
| `ImportError: Missing optional dependency 'openpyxl'` | `pip install openpyxl` |
| Script lentísimo | bucles con `iterrows()`: usar operaciones sobre columnas enteras (vectorizadas) |
