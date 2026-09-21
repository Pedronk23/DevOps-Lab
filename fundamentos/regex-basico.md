# Fundamentos — Expresiones regulares (regex)

Una **regex** es una expresión regular, usada para buscar o validar texto.

## Ejemplo desglosado

```regex
^([A-Za-z]+)
```

Captura una o más letras al principio del texto.

| Elemento | Significado |
|---|---|
| `^` | inicio del texto |
| `( )` | grupo de captura |
| `[A-Za-z]` | cualquier letra mayúscula o minúscula |
| `+` | una o más veces |

## Resultados

| Entrada | Captura |
|---|---|
| `Pedro123` | `Pedro` |
| `ABC.03.26.2` | `ABC` |

## Otros elementos útiles

| Elemento | Significado |
|---|---|
| `$` | fin del texto |
| `.` | cualquier carácter |
| `*` | cero o más veces |
| `?` | cero o una vez |
| `\d` | dígito |
| `\w` | carácter de palabra (letra, dígito o `_`) |
| `{n,m}` | entre n y m repeticiones |

## Chuleta completa

### Clases de caracteres

| Elemento | Significado |
|---|---|
| `[abc]` | uno de esos caracteres |
| `[^abc]` | cualquiera **excepto** esos |
| `[a-z0-9]` | rangos |
| `\d` / `\D` | dígito / no dígito |
| `\w` / `\W` | carácter de palabra / lo contrario |
| `\s` / `\S` | espacio en blanco (espacio, tab, salto de línea) / lo contrario |
| `.` | cualquier carácter salvo el salto de línea |

> **Cuidado con `[A-z]`**: no equivale a `[A-Za-z]`. Entre la `Z` y la `a` de la tabla ASCII
> hay otros caracteres (`[`, `\`, `]`, `^`, `_` y el acento grave), que también entrarían.

### Anclas

| Elemento | Significado |
|---|---|
| `^` | inicio (de texto o de línea, según el modo) |
| `$` | fin |
| `\b` | límite de palabra: `\bweb\b` casa con "web" pero no con "website" |

### Cuantificadores

| Elemento | Significado |
|---|---|
| `*` | 0 o más |
| `+` | 1 o más |
| `?` | 0 o 1 (opcional) |
| `{3}` | exactamente 3 |
| `{2,}` | 2 o más |
| `{2,5}` | entre 2 y 5 |
| `*?`, `+?` | versión **perezosa**: lo mínimo posible |

### Grupos y alternativas

| Elemento | Significado |
|---|---|
| `(abc)` | grupo de captura (se referencia con `\1` o `$1`) |
| `(?:abc)` | grupo **sin** captura: solo para agrupar |
| `(?<nombre>abc)` | grupo con nombre (en Python: `(?P<nombre>abc)`) |
| `a\|b` | a **o** b |
| `(?=abc)` | *lookahead*: seguido de abc, sin consumirlo |
| `(?!abc)` | *lookahead negativo*: **no** seguido de abc |
| `(?<=abc)` | *lookbehind*: precedido de abc, sin consumirlo |
| `(?<!abc)` | *lookbehind negativo*: **no** precedido de abc |
| `\1` … `\9` | *backreference*: repite lo que capturó ese grupo |
| `\k<nombre>` | backreference a un grupo con nombre |

### Modificadores (flags)

| Flag | Efecto |
|---|---|
| `i` | ignora mayúsculas/minúsculas |
| `m` | multilínea: `^` y `$` casan en cada línea |
| `s` | el `.` también casa con el salto de línea |
| `g` | global: todas las coincidencias, no solo la primera (JavaScript, sed) |
| `u` | unicode: habilita `\u{...}` para cualquier carácter (JavaScript) |

### Caracteres que hay que escapar

Para buscarlos literalmente, van con `\` delante:

```
.  *  +  ?  ^  $  (  )  [  ]  {  }  |  \  /
```

```regex
web01\.lab\.local        ← sin escapar, "." casaría con cualquier carácter: "web01XlabYlocal"
```

## Codicioso vs perezoso

```
   Texto:     <a><b>

   <.+>       →  <a><b>     codicioso: se lleva todo lo que puede
   <.+?>      →  <a>        perezoso: para en cuanto puede
```

## Recetas para DevOps

| Qué | Regex |
|---|---|
| IPv4 (aproximada: acepta 999.1.1.1) | `\b(?:\d{1,3}\.){3}\d{1,3}\b` |
| Versión semántica | `^v?(\d+)\.(\d+)\.(\d+)$` |
| Fecha ISO | `\d{4}-\d{2}-\d{2}` |
| Nivel de log | `\b(ERROR\|WARN)\b` |
| Nombre válido en Kubernetes (DNS-1123) | `^[a-z0-9]([-a-z0-9]*[a-z0-9])?$` |
| Línea vacía o solo espacios | `^\s*$` |
| Línea comentada | `^\s*#` |
| Espacios al final de línea | `\s+$` |
| Hosts web01…web99 | `\bweb\d{2}\b` |
| Contraseña con al menos un dígito, una mayúscula y 12 caracteres | `^(?=.*\d)(?=.*[A-Z]).{12,}$` |

> Para validar de verdad IPs, emails o URLs, mejor una librería que una regex casera.

Un nombre DNS completo, con etiquetas de 1 a 63 caracteres que no empiezan ni acaban en guion
(usa lookahead y lookbehind a la vez):

```regex
^(?!-)[a-z0-9-]{1,63}(?<!-)(?:\.(?!-)[a-z0-9-]{1,63}(?<!-))*$
```

Palabras repetidas ("el el perro"), con backreference:

```regex
\b(\w+)\s+\1\b
```
> Las regex "perfectas" de email ocupan varias líneas y aun así fallan en casos raros.

## Sabores: no todas las regex son iguales

| Sabor | Dónde | Diferencias clave |
|---|---|---|
| **BRE** (básica) | `grep`, `sed` por defecto | `+ ? \| ( ) { }` son **literales**; hay que escribir `\+`, `\(…\)` |
| **ERE** (extendida) | `grep -E`, `sed -E`, `awk` | `+ ? \| ( )` funcionan sin escapar. **Sin `\d`**: usar `[0-9]` |
| **PCRE** | `grep -P`, Perl, PHP, nginx | lo tiene todo: `\d`, lookahead, perezosos |
| **Python `re`** | Python, Ansible | como PCRE; grupos con nombre `(?P<n>…)` |
| **.NET** | PowerShell | como PCRE; grupos con nombre `(?<n>…)`; sin distinguir mayúsculas por defecto |
| **RE2** | Go, Prometheus, Grafana Loki | sin lookahead ni referencias atrás (a cambio, nunca se "cuelga") |

```bash
echo "abc 123" | grep -oE '\d+'        # nada: ERE no conoce \d
echo "abc 123" | grep -oE '[0-9]+'     # 123
echo "abc 123" | grep -oP '\d+'        # 123
```

> En **Prometheus** las regex de los selectores están **ancladas**: `instance=~"web"` solo
> casa con el texto exacto `web`. Para "contiene web" hay que escribir `instance=~".*web.*"`.

## Usarlas en cada herramienta

### grep y sed

```bash
grep -E '\b(ERROR|WARN)\b' app.log
grep -oP '(?<=user=)\w+' auth.log              # solo lo que va detrás de "user="
grep -vE '^\s*(#|$)' /etc/ssh/sshd_config      # sin comentarios ni vacías

sed -E 's/^(\w+)\.lab\.local$/\1/' hosts.txt   # web01.lab.local → web01
```

Más en [sed](../linux-shell/05-sed.md).

### PowerShell

```powershell
'web01.lab.local' -match '^(?<host>\w+)\.lab\.local$'   # True
$Matches.host                                             # web01

Select-String -Path .\logs\*.log -Pattern 'ERROR|WARN'   # el "grep" de PowerShell

'2026-09-14' -replace '(\d{4})-(\d{2})-(\d{2})', '$3/$2/$1'   # 14/09/2026
```

> Con `-replace`, el reemplazo va entre **comillas simples**: con dobles, PowerShell intenta
> expandir `$1` como variable y queda vacío.

### Python

```python
import re

m = re.search(r"(?P<anio>\d{4})-(?P<mes>\d{2})-(?P<dia>\d{2})", "fecha 2026-09-14")
m.group("anio")                                   # '2026'
re.findall(r"\b(?:\d{1,3}\.){3}\d{1,3}\b", texto) # todas las IPs
re.sub(r"\s+$", "", linea)                        # quitar espacios finales
```

Usa siempre cadenas `r"..."` (*raw*) para no tener que duplicar las `\`.

### Ansible

```yaml
- name: Extraer la versión
  ansible.builtin.set_fact:
    version: "{{ salida.stdout | regex_search('v(\\d+\\.\\d+\\.\\d+)', '\\1') | first }}"

- name: Solo en hosts web
  ansible.builtin.debug:
    msg: "es un servidor web"
  when: inventory_hostname is match('web\\d+')
```

(dentro de strings YAML con comillas dobles, cada `\` va doble)

### JavaScript (Node.js)

```js
const re1 = /error/g;                    // literal
const re2 = new RegExp("error", "g");    // constructor: para patrones dinámicos

"2026-09-21".match(/(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/).groups.year  // "2026"
/\b(\w+)\s+\1\b/.test("el el perro")   // true  (backreference)
"Mr. Smith".match(/(?<=Mr\.\s)\w+/)[0]   // "Smith"  (lookbehind)
"100€ 200$".match(/\d+(?=€)/)[0]          // "100"  (lookahead)
/\u{2200}/u.test("∀")                     // true  (requiere el flag u)
```

Un CSV sencillo a objetos, partiendo por líneas con `/\r?\n/` (así vale igual con finales de
línea de Windows y de Linux):

```js
const csv = `nombre,edad
Ana,30
Luis,25`;

const [cabecera, ...filas] = csv.trim().split(/\r?\n/);
const claves = cabecera.split(",");

const json = filas.map(fila =>
  Object.fromEntries(fila.split(",").map((v, i) => [claves[i], v]))
);
// [{ nombre: "Ana", edad: "30" }, { nombre: "Luis", edad: "25" }]
```

> Para trocear un CSV de verdad (comillas, comas dentro de un campo, saltos de línea) no uses
> regex: usa una librería, como [pandas](../python/pandas.md) o el módulo `csv` de Python.

## Probar antes de usar

- **regex101.com**: explica cada parte, permite elegir el sabor (PCRE, Python, Go…) y probar
  con texto real. Ojo con pegar logs con datos sensibles.
- En consola: `grep` contra un fichero de ejemplo antes de lanzar un `sed -i`.

## Cuidado: backtracking catastrófico

Patrones con cuantificadores anidados como `(a+)+$` o `(.*)*` pueden tardar **segundos o
minutos** con ciertos textos, porque el motor prueba combinaciones exponenciales. Es una vía
de denegación de servicio (ReDoS) cuando la regex procesa entrada de usuarios. Solución:
patrones más específicos, o motores tipo RE2 que no hacen backtracking.
