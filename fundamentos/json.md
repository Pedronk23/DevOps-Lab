# Fundamentos — JSON

**JSON** = *JavaScript Object Notation*. Formato de texto ligero que se utiliza para almacenar
e intercambiar datos. Es el idioma universal del intercambio de datos.

## ¿Para qué se usa?

- Enviar datos entre un servidor y una app web o móvil (ej.: al abrir el teléfono).
- Guardar configuraciones de programas y videojuegos.
- Almacenar información en BBDD modernas (NoSQL, tipo Firebase o MongoDB).

Y en DevOps, continuamente:

- Respuestas de **APIs REST** (GitLab, AWX, Nexus, Vault, la API de Kubernetes).
- Salida de CLIs: `kubectl -o json`, `docker inspect`, `az … -o json`, `terraform show -json`.
- Configuración: `daemon.json` de Docker, `package.json`, `settings.json` de VS Code.
- Logs estructurados (una línea JSON por evento).

## Estructura básica: clave-valor

Estructura la información en parejas `"clave": valor` envueltas en llaves.

- **Clave**: es el nombre del dato (siempre entre comillas dobles).
- **Valor**: es el dato real.

```json
{
  "nombre": "Carlos",
  "edad": 28,
  "es_premium": true,
  "hobbies": ["leer", "programar"],
  "direccion": {
    "ciudad": "Madrid",
    "codigo_postal": "28001"
  }
}
```

```
   {                      ← objeto
     "hobbies": [         ← array (lista ordenada)
        "leer",           ← string
        "programar"
     ],
     "direccion": {       ← objeto anidado
        "ciudad": …
     }
   }
   ruta para llegar a la ciudad:   .direccion.ciudad
   ruta al primer hobby:            .hobbies[0]
```

## Tipos de dato que acepta

| Tipo | Detalle |
|---|---|
| Texto (string) | siempre entre comillas dobles (`"Carlos"`) |
| Números (number) | sin comillas, enteros o decimales (`28`, `1.75`) |
| Booleanos | valores lógicos: `true` o `false` |
| Listas (arrays) | conjunto de elementos ordenados, entre corchetes `[ ]` |
| Objetos | un JSON dentro de otro JSON, entre llaves `{ }` |
| Nulo (null) | para indicar que un campo está vacío |

Lo que **no** existe en JSON: fechas (se usan strings ISO 8601, `"2026-09-14T10:22:00Z"`),
comentarios, `undefined`, `NaN` o `Infinity`.

> `codigo_postal` va entre comillas a propósito: como número, `"08001"` perdería el cero
> inicial. Los números JSON **no admiten ceros a la izquierda**.

## Las 3 reglas de oro

1. **No uses comillas simples**: claves y textos siempre con comillas dobles.
2. **Cuidado con las comas**: cada pareja clave-valor se separa con una coma, pero el último
   elemento de la lista o del objeto **no lleva coma**.
3. **No acepta comentarios**: no puedes poner notas con `//` ni `/* */`.

> Excepción que confunde: `settings.json` de VS Code y `tsconfig.json` son **JSONC** (JSON
> con comentarios). Es un dialecto: un parser JSON estricto los rechaza.

## Caracteres especiales (escapes)

| Escribir | Para obtener |
|---|---|
| `\"` | comillas dobles dentro de un string |
| `\\` | una barra invertida |
| `\n` / `\t` | salto de línea / tabulador |
| `\u00f1` | `ñ` (cualquier carácter Unicode) |

```json
{ "ruta": "C:\\apps\\MiAppWeb", "mensaje": "Dijo \"hola\"\nY se fue" }
```

> Las rutas de Windows son la fuente número uno de JSON inválido: cada `\` debe ir doble.

## JSON, YAML y XML

```json
{"app": "web", "replicas": 3, "puertos": [80, 443]}
```

```yaml
app: web
replicas: 3
puertos:
  - 80
  - 443
```

```xml
<app nombre="web"><replicas>3</replicas><puertos><p>80</p><p>443</p></puertos></app>
```

| | JSON | YAML | XML |
|---|---|---|---|
| Legibilidad humana | media | alta | baja |
| Comentarios | no | sí (`#`) | sí |
| Estructura por | llaves y corchetes | **indentación** | etiquetas |
| Tipos | pocos y estrictos | muchos (y a veces sorprendentes) | todo es texto |
| Uso típico | APIs, máquinas | configuración escrita por personas (Kubernetes, Ansible) | SOAP, Maven, sistemas legados |

> **Todo JSON válido es YAML válido** (YAML 1.2). Por eso `kubectl apply -f` acepta
> manifiestos en JSON sin más.

## jq: la navaja suiza de JSON en consola

```bash
sudo apt install jq        # o: winget install jqlang.jq
```

| Filtro | Qué hace |
|---|---|
| `jq .` | formatea y colorea |
| `jq -c .` | compacto, en una línea |
| `jq -r '.campo'` | valor en **crudo** (sin comillas): lo que quieres en scripts |
| `.direccion.ciudad` | navegar |
| `.hobbies[0]` / `.hobbies[]` | un elemento / todos |
| `.items \| length` | cuántos hay |
| `keys` | claves de un objeto |
| `select(condición)` | filtrar |
| `map(.campo)` | transformar cada elemento de un array |
| `{nombre: .metadata.name}` | construir un objeto nuevo |
| `@csv` / `@tsv` | exportar filas |
| `--arg nombre valor` | pasar una variable del shell |

### Recetas con kubectl

```bash
# nombres de todos los pods
kubectl get pods -A -o json | jq -r '.items[].metadata.name'

# pods que no están Running, con su namespace
kubectl get pods -A -o json | jq -r '
  .items[]
  | select(.status.phase != "Running")
  | "\(.metadata.namespace)/\(.metadata.name)  \(.status.phase)"'

# imágenes en uso, sin repetir
kubectl get pods -A -o json | jq -r '.items[].spec.containers[].image' | sort -u

# de una API: jobs fallidos de AWX
curl -s -H "Authorization: Bearer $TOKEN" "https://awx.lab.local/api/v2/jobs/?status=failed" \
  | jq -r '.results[] | [.id, .name, .finished] | @tsv'

# variable del shell dentro del filtro
jq --arg entorno prod '.[] | select(.entorno == $entorno) | .hostname' inventario.json

# modificar y guardar
jq '.["registry-mirrors"] = ["https://dockerhub-proxy.lab.local"]' daemon.json > daemon.json.new
```

> Para una consulta sencilla, `kubectl` tiene su propio lenguaje, **jsonpath**:
> `kubectl get pods -o jsonpath='{.items[*].metadata.name}'`. Para cualquier cosa con
> filtros o formato, jq es más cómodo.

## JSON en otros lenguajes

```powershell
# PowerShell
$cfg = Get-Content .\config.json -Raw | ConvertFrom-Json
$cfg.direccion.ciudad
$cfg.edad = 29
$cfg | ConvertTo-Json -Depth 10 | Set-Content .\config.json
Invoke-RestMethod https://api.lab.local/v1/apps     # convierte la respuesta JSON en objetos
```

> `ConvertTo-Json` tiene **profundidad 2 por defecto**: los objetos más anidados se
> convierten en el texto `"System.Collections.Hashtable"`. Pon siempre `-Depth`.

```python
# Python
import json

with open("config.json", encoding="utf-8") as f:
    cfg = json.load(f)

cfg["edad"] = 29

with open("config.json", "w", encoding="utf-8") as f:
    json.dump(cfg, f, indent=2, ensure_ascii=False)   # ensure_ascii=False: guarda "ñ" y no "\u00f1"
```

```yaml
# Ansible
- name: Leer la salida JSON de un comando
  ansible.builtin.command: kubectl get nodes -o json
  register: nodos
  changed_when: false

- name: Mostrar los nombres
  ansible.builtin.debug:
    msg: "{{ (nodos.stdout | from_json)['items'] | map(attribute='metadata.name') | list }}"
```

## Validar

```bash
jq empty fichero.json && echo "válido"      # no imprime nada si es válido
python -m json.tool fichero.json            # formatea, o dice línea y columna del error
```

Para validar además la **estructura** (que existan ciertos campos, con cierto tipo) existe
**JSON Schema**: un JSON que describe cómo debe ser otro JSON. Lo usan los editores para
autocompletar y validar ficheros de configuración.

## JSON Lines (logs estructurados)

Un objeto JSON **por línea**, sin corchetes ni comas entre ellos:

```
{"ts":"2026-09-14T10:22:01Z","level":"info","msg":"arranque","puerto":8080}
{"ts":"2026-09-14T10:22:05Z","level":"error","msg":"no conecta a la BBDD","intento":1}
```

```bash
jq -c 'select(.level == "error")' app.log      # jq procesa línea a línea
```

Es el formato que prefieren Loki, Elasticsearch y compañía: cada línea se puede parsear sola.

## Errores típicos

| Error | Causa |
|---|---|
| `Unexpected token '` / `Expecting property name enclosed in double quotes` | comillas simples, o una clave sin comillas |
| `Unexpected token }` / `Expecting value` | **coma final** después del último elemento |
| `Unexpected token /` | comentarios |
| `Invalid \escape` | ruta de Windows con `\` simple |
| `Unexpected token` en la posición 0 con un fichero que "se ve bien" | **BOM** al principio del fichero (ver [codificación](parsers-y-codificacion.md)) |
| `parse error: Invalid numeric literal` en jq | lo que llega no es JSON: un HTML de error, un 401, un mensaje del proxy |
