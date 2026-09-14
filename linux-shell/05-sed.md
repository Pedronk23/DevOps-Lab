# Linux — sed (Stream Editor)

En Linux, `sed` significa *Stream Editor* (editor de flujo). Una de las herramientas más
potentes y clásicas, utilizada para buscar, filtrar, modificar o transformar texto de forma
automática, sin necesidad de abrir el fichero en un editor (como nano o vim).

Toma un flujo de datos (un archivo o la salida de otro comando), realiza las modificaciones
según las instrucciones que le des y muestra el resultado en pantalla (o lo guarda en un
archivo).

## Cómo procesa

```
   fichero / stdin
        │  línea 1, línea 2, …
        ▼
   ┌────────────────────────────────────────┐
   │ para CADA línea:                       │
   │   1. la carga en el "pattern space"    │
   │   2. aplica los comandos cuya          │
   │      dirección coincide                │
   │   3. imprime el resultado (salvo -n)   │
   └────────────────────────────────────────┘
        │
        ▼
   stdout  (o el propio fichero con -i)
```

## Usos

| Uso | Detalle |
|---|---|
| **Reemplazar texto** | cambiar una palabra por otra en todo un archivo |
| **Eliminar líneas** | borrar líneas específicas o las que cumplan un patrón |
| **Buscar y filtrar** | mostrar solo las líneas que contienen cierta palabra |
| **Automatización** | modificar archivos masivamente mediante scripts |

## Estructura

```bash
sed [opciones] 'comando' archivo
```

> Por defecto `sed` **no modifica el archivo original**: solo lee el contenido, lo cambia en
> memoria y muestra el resultado en el terminal. Si quieres aplicar los cambios hay que usar
> `-i` (*in-place*).

## Ejemplo

```bash
sed 's/error/correcto/g' log.txt
```

| Parte | Significado |
|---|---|
| `s` | indica sustitución |
| `error` | palabra que buscas |
| `correcto` | palabra por la que la vas a cambiar |
| `g` | modificador global: cambia todas las apariciones en cada línea |

Sin la `g`, solo cambia **la primera** aparición de cada línea.

## Opciones

**`-i`** — modifica el archivo original directamente.

```bash
sed -i 's/hola/adios/g' archivo.txt   # guarda los cambios
sed -i.bak 's/hola/adios/g' archivo.txt   # guarda los cambios y deja copia en archivo.txt.bak
```

**`-n`** — solo imprime lo que le decimos en específico.

```bash
sed -n '5p' archivo.txt               # solo muestra la línea 5
```

**`-e`** — encadenar múltiples expresiones en una sola línea, sin usar pipes.

```bash
sed -e 's/azul/rojo/g' -e 's/verde/amarillo/g' colores.txt
```

**`-E`** — expresiones regulares extendidas: `+`, `?`, `|`, `()` sin escapar.

```bash
sed -E 's/([0-9]+)\.([0-9]+)/\2.\1/' versiones.txt
```

| Opción | Qué hace |
|---|---|
| `-i` / `-i.bak` | edita el fichero / con copia de seguridad |
| `-n` | no imprime nada salvo lo que pidas con `p` |
| `-e` | añade una expresión |
| `-E` | regex extendida (ver [regex](../fundamentos/regex-basico.md)) |
| `-f script.sed` | lee los comandos de un fichero |
| `-s` | trata varios ficheros por separado (para rangos y `$`) |

> **macOS/BSD**: `sed -i` exige un argumento de extensión: `sed -i '' 's/a/b/' fichero`.
> En GNU/Linux eso fallaría. Si un script tiene que funcionar en ambos, usa `-i.bak` y borra
> el `.bak` después.

## Comandos dentro de las comillas

### `s` — buscar un patrón y cambiarlo por otro

| Flag | Significado |
|---|---|
| `/g` | *global*: cambia todas las veces que aparece la palabra |
| `/I` | *insensitive*: la búsqueda ignora mayúsculas y minúsculas |
| `/2` | solo la segunda aparición de cada línea |
| `/p` | imprime la línea si hubo sustitución (con `-n`) |

Caracteres especiales en el reemplazo:

| Símbolo | Significado |
|---|---|
| `&` | todo el texto que ha coincidido |
| `\1` … `\9` | el contenido del grupo de captura 1…9 |

```bash
echo "puerto 8080" | sed 's/[0-9]\+/[&]/'           # puerto [8080]
echo "pedro garcia" | sed -E 's/(\w+) (\w+)/\2, \1/' # garcia, pedro
```

**Otro delimitador**: cuando el patrón tiene `/` (rutas, URLs), usa `|`, `#` o `:`:

```bash
sed 's|/usr/local/bin|/opt/bin|g' script.sh
sed 's#http://#https://#g' config.yml
```

### `d` — borrar líneas completas

```bash
sed '3d' archivo.txt          # línea específica
sed '1,10d' archivo.txt       # rango de líneas
sed '/^#/d' config.conf       # por patrón: todas las líneas que empiecen por #
```

### `p` — print

Se usa con `-n` para ver solo lo que interesa.

```bash
sed -n '/ERROR/p' log.txt     # solo muestra las líneas con la palabra ERROR
```

> `sed '/ERROR/p'` **sin** `-n` imprime esas líneas **dos veces** (la impresión normal más
> la del `p`).

### Otros comandos

| Comando | Qué hace | Ejemplo |
|---|---|---|
| `i\` | inserta una línea **antes** | `sed '/^\[server\]/i\# sección gestionada' app.ini` |
| `a\` | añade una línea **después** | `sed '/^\[server\]/a\port = 8080' app.ini` |
| `c\` | reemplaza la línea entera | `sed '/^PermitRootLogin/c\PermitRootLogin no' sshd_config` |
| `y` | traduce carácter a carácter | `sed 'y/abc/ABC/'` |
| `q` | sale (deja de leer) | `sed '10q' enorme.log` (como `head`) |
| `=` | imprime el número de línea | `sed -n '/ERROR/=' app.log` |
| `!` | niega la dirección | `sed '/^#/!d'` → deja solo los comentarios |

## Direcciones: sobre qué líneas actuar

| Dirección | Aplica a |
|---|---|
| `5` | la línea 5 |
| `$` | la última línea |
| `3,8` | de la 3 a la 8 |
| `5,$` | de la 5 al final |
| `/patron/` | las líneas que coinciden |
| `/inicio/,/fin/` | desde la que casa con "inicio" hasta la que casa con "fin" |
| `0~2` | líneas pares (GNU) |

```bash
sed -n '10,20p' app.log                        # líneas 10 a 20
sed -n '/BEGIN CERTIFICATE/,/END CERTIFICATE/p' fullchain.pem   # extraer bloques PEM
sed '/^\[database\]/,/^\[/ s/^host=.*/host=db01/' app.ini       # cambiar solo dentro de una sección
sed '$d' fichero                               # borrar la última línea
```

## Recetas DevOps

```bash
# ver un fichero de configuración sin comentarios ni líneas vacías
sed -e '/^\s*#/d' -e '/^\s*$/d' /etc/ssh/sshd_config

# descomentar y fijar una directiva
sudo sed -i -E 's/^\s*#?\s*PermitRootLogin\s+.*/PermitRootLogin no/' /etc/ssh/sshd_config

# cambiar la etiqueta de una imagen en un manifiesto (variables → comillas DOBLES)
TAG=1.4.2
sed -i "s|image: registry.lab.local/app:.*|image: registry.lab.local/app:${TAG}|" deploy.yaml

# reemplazar en muchos ficheros a la vez
grep -rl 'nexus.old.local' . | xargs sed -i 's/nexus\.old\.local/nexus.lab.local/g'

# quitar los finales de línea de Windows (CRLF) → evita "/bin/bash^M: bad interpreter"
sed -i 's/\r$//' script.sh

# quitar el BOM UTF-8 del principio de un fichero (GNU sed)
sed -i '1s/^\xEF\xBB\xBF//' config.json

# borrar espacios al final de línea
sed -i 's/[[:space:]]*$//' fichero

# añadir una línea al final si no existe
grep -q '^vm.max_map_count' /etc/sysctl.conf || echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.conf
```

> Con variables del shell dentro del patrón, **comillas dobles**. Con comillas simples,
> `${TAG}` llega a sed tal cual, como texto literal.

## sed, grep y awk: cuál usar

| Herramienta | Mejor para |
|---|---|
| `grep` | **buscar**: ¿qué líneas contienen esto? |
| `sed` | **transformar** líneas: sustituir, borrar, insertar |
| `awk` | **columnas y cálculos**: "suma la columna 3", "imprime el campo 1 si el 5 > 100" |
| `yq` / `jq` | **YAML / JSON**: modificar estructuras sin romper la sintaxis |

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head   # top de IPs
yq -i '.spec.replicas = 3' deploy.yaml                           # mejor que sed para YAML
```

## Buenas prácticas

- Prueba **sin `-i`** primero y mira la salida; después añade `-i`.
- En ficheros importantes, `-i.bak` o comprobar con `diff` antes de borrar la copia.
- Si la sustitución no cambia nada, sed **no da error**: en scripts verifica después con
  `grep` que el cambio está.
- Para editar configuración de forma repetible y con control de cambios, mejor Ansible
  (`lineinfile`, `replace`, `template`) que sed en un script suelto.
