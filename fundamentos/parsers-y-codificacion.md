# Fundamentos — Parsers y codificación de texto

## Parser

Es un código que:

1. Lee texto.
2. Entiende su estructura.
3. Lo convierte en datos utilizables.

Ejemplo: un parser de YAML convierte esto…

```yaml
nombre: Pedro
edad: 25
```

…en una estructura de datos de Python:

```python
{
    "nombre": "Pedro",
    "edad": 25
}
```

## Qué pasa por dentro

```
   texto  ──►  LEXER  ──────────►  PARSER  ────────────►  estructura de datos
               (tokens)              (gramática)             (dict, lista, AST)

   "edad: 25"   [CLAVE "edad"]      "un mapa con una         {"edad": 25}
                [DOS_PUNTOS]         clave edad cuyo
                [ENTERO 25]          valor es 25"
```

| Término | Significado |
|---|---|
| **Lexer / tokenizer** | parte el texto en piezas con significado (palabras, números, símbolos) |
| **Parser** | comprueba que las piezas siguen la gramática y construye la estructura |
| **AST** | *Abstract Syntax Tree*: árbol que representa la estructura (así funcionan Jinja2, compiladores, linters) |
| **Deserializar** | texto → datos (lo que hace un parser) |
| **Serializar** | datos → texto (`json.dumps`, `to_yaml`, `ConvertTo-Json`) |

### Parsers con los que tratas sin darte cuenta

| Formato | Parser / quién lo usa |
|---|---|
| YAML | PyYAML (Ansible), `sigs.k8s.io/yaml` (Kubernetes), `yq` |
| JSON | `jq`, `ConvertFrom-Json`, `json` de Python |
| HCL | Terraform, Vault/OpenBao (policies y configuración) |
| Jinja2 | Ansible: primero se renderiza la plantilla, **después** se parsea el YAML resultante |
| INI / TOML | `ansible.cfg` (INI), `pyproject.toml`, configuración de Rust y Go |
| XML | Maven (`pom.xml`), Tomcat (`server.xml`), MSBuild |

> Cuando un error dice `line 12, column 5`, lo dice el **parser**: el problema está en la
> sintaxis, no en la lógica. Y a veces la línea real del fallo es la anterior (una comilla
> o un corchete sin cerrar).

### Trampas de YAML: el parser "interpreta" más de lo que esperas

```yaml
pais: NO          # YAML 1.1 (PyYAML) → false          (el famoso "Norway problem")
activo: yes       # YAML 1.1 → true
permisos: 0755    # YAML 1.1 → 493 (entero en octal), no el texto "0755"
version: 1.10     # → 1.1 (número decimal): se come el cero
hora: 22:22       # YAML 1.1 → 1342 (¡base 60!)
nulo: ~           # → null
```

La solución es siempre la misma: **entrecomillar** lo que deba ser texto.

```yaml
pais: "NO"
permisos: "0755"
version: "1.10"
```

En Ansible, `mode: "0644"` entre comillas por esta razón (ver
[YAML e idempotencia](../ansible/08-yaml-e-idempotencia.md)).

## Codificación de texto

Un fichero son **bytes**. La codificación es el acuerdo de qué carácter representa cada byte.

```
   carácter  "ñ"
       │
       ├── UTF-8         → C3 B1          (2 bytes)
       ├── ISO-8859-1    → F1             (1 byte)
       ├── Windows-1252  → F1             (1 byte)
       └── UTF-16 LE     → F1 00          (2 bytes)
```

| Codificación | Notas |
|---|---|
| **ASCII** | 7 bits, 128 caracteres: letras inglesas, números y símbolos. Sin tildes ni `ñ` |
| **ISO-8859-1 (Latin-1)** | 1 byte, idiomas de Europa occidental |
| **Windows-1252 (cp1252)** | la "ANSI" de Windows en español; casi igual que Latin-1 (añade `€`, comillas tipográficas…) |
| **UTF-8** | 1 a 4 bytes por carácter, **compatible con ASCII**, cubre todo Unicode. El estándar de facto |
| **UTF-16 LE** | 2 o 4 bytes; la que usa Windows internamente (y PowerShell 5.1 al redirigir con `>`) |

## UTF-8 sin BOM

### UTF-8

- Es una **codificación de texto**.
- Define cómo se guardan los caracteres en bytes para que el ordenador los entienda.
- Permite guardar: letras, tildes, `ñ`, emojis, caracteres chinos/japoneses, símbolos…

### BOM (Byte Order Mark)

- Pequeña marca **invisible** que se pone al inicio del archivo para indicar que es UTF-8.
- Son bytes especiales al principio del archivo (`EF BB BF`).
- **Puede dar errores y fallos** si existe en: scripts de Python, parsers, Linux, Dockerfiles,
  JSON, YAML, compiladores y terminales.

> Por eso la convención en entornos Linux/DevOps es **guardar siempre en UTF-8 sin BOM**.
> Windows (Notepad, algunos editores y PowerShell antiguos) lo añade por defecto: cuidado al
> generar ficheros de configuración desde Windows.

| BOM | Codificación |
|---|---|
| `EF BB BF` | UTF-8 |
| `FF FE` | UTF-16 LE |
| `FE FF` | UTF-16 BE |

Síntomas típicos del BOM:

```
/bin/bash: #!/bin/bash: No such file or directory       ← el shebang no empieza por "#!"
json.decoder.JSONDecodeError: Unexpected UTF-8 BOM
Error parsing JSON: Unexpected token  in JSON at position 0
```

## Mojibake: cuando se lee con la codificación equivocada

```
   Se escribió "año" en UTF-8   →  bytes 61 C3 B1 6F
   Se leyó como Latin-1         →  "aÃ±o"

   Se escribió "año" en cp1252  →  bytes 61 F1 6F
   Se leyó como UTF-8           →  "a�o"   (F1 no es una secuencia UTF-8 válida)
```

| Ves | Pasó |
|---|---|
| `Ã±`, `Ã©`, `Ã³` | UTF-8 interpretado como Latin-1/cp1252 |
| `�` | Latin-1/cp1252 interpretado como UTF-8 |
| `ï»¿` al principio | un BOM UTF-8 mostrado como Latin-1 |

## Finales de línea: CRLF vs LF

| Sistema | Fin de línea | Bytes |
|---|---|---|
| Linux / macOS | `LF` (`\n`) | `0A` |
| Windows | `CRLF` (`\r\n`) | `0D 0A` |

```
$ ./deploy.sh
/bin/bash^M: bad interpreter: No such file or directory     ← script guardado con CRLF
```

```bash
sed -i 's/\r$//' deploy.sh           # quitar los CR
dos2unix deploy.sh                   # lo mismo, si está instalado
```

En Git, para que no vuelva a pasar, un `.gitattributes` en la raíz del repositorio:

```
* text=auto eol=lf
*.ps1 text eol=crlf
*.bat text eol=crlf
*.png binary
```

## Detectar y convertir

```bash
file config.json              # "... UTF-8 (with BOM) text, with CRLF line terminators"
head -c 3 config.json | xxd   # 00000000: efbb bf   → hay BOM
grep -rl $'\xEF\xBB\xBF' .    # ficheros con BOM en el árbol

sed -i '1s/^\xEF\xBB\xBF//' config.json                       # quitar BOM
iconv -f WINDOWS-1252 -t UTF-8 exportado.csv > exportado-utf8.csv   # convertir
```

En **VS Code**, la barra de estado muestra la codificación (`UTF-8`, `UTF-8 with BOM`) y el
fin de línea (`LF`/`CRLF`); pulsando encima se puede **reabrir** o **guardar** con otra.

## PowerShell y la codificación

| Operación | Windows PowerShell 5.1 | PowerShell 7 |
|---|---|---|
| `>` y `Out-File` | **UTF-16 LE con BOM** | UTF-8 sin BOM |
| `Set-Content` / `Add-Content` | ANSI (cp1252 en español) | UTF-8 sin BOM |
| `-Encoding UTF8` | UTF-8 **con** BOM | UTF-8 sin BOM (`utf8BOM` si lo quieres) |
| `-Encoding utf8NoBOM` | no existe | disponible |

UTF-8 **sin** BOM desde Windows PowerShell 5.1:

```powershell
$utf8SinBom = New-Object System.Text.UTF8Encoding $false
[IO.File]::WriteAllText("C:\despliegue\config.json", $contenido, $utf8SinBom)
```

> Un script de 5.1 que genera un YAML o un JSON con `>` produce un fichero UTF-16: en Linux
> se ve con un espacio entre cada letra (`h o l a`) y ningún parser lo acepta.

## Python y la codificación

```python
open("config.yml", encoding="utf-8")        # indicar SIEMPRE la codificación
open("exportado.csv", encoding="utf-8-sig") # lee UTF-8 y se come el BOM si lo hay
open("legado.txt", encoding="cp1252")       # ficheros antiguos de Windows
```

Sin `encoding=`, Python usa la del sistema: en Windows puede ser cp1252 y el mismo script
funciona en Linux y falla en Windows.

## Resumen para no complicarse

- Ficheros de configuración, scripts y código: **UTF-8 sin BOM y LF**.
- Scripts `.ps1` que tienen tildes y deben ejecutarse en **Windows PowerShell 5.1**: UTF-8
  **con** BOM (sin BOM, 5.1 los lee como ANSI y rompe las tildes).
- CSV para abrir en Excel: UTF-8 con BOM (`utf-8-sig`) o Excel mostrará mojibake.
- Declara la codificación siempre que el lenguaje lo permita.
