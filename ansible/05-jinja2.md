# Jinja2

## Qué es

Motor de plantillas (*template engine*) desarrollado para Python. Permite **combinar texto
estático con datos dinámicos** para generar un resultado automáticamente.

La idea de fondo: en vez de crear un archivo de configuración a mano para cada uno de los
N destinos (cientos de servidores o clientes), se genera uno dinámico con variables.

## Ejemplo mínimo

Plantilla `.j2`:

```jinja
Servidor: {{ hostname }}
IP: {{ ip }}
```

Variables (YAML):

```yaml
hostname: web01
ip: 10.0.0.15
```

Resultado:

```
Servidor: web01
IP: 10.0.0.15
```

## Conceptos

- **Renderizar**: sustituir las expresiones dinámicas por valores reales para producir el
  documento final.
- Extensión: `.j2` → `nginx.conf.j2`, `config.json.j2`, …
- El contenido puede ser cualquier texto: YAML, JSON, HTML, XML, scripts, configs.

## Ansible con Jinja2

Ansible usa Jinja2 como sistema de templating interno.

```yaml
- name: Generar configuración
  ansible.builtin.template:
    src: archivo.conf.j2
    dest: /etc/archivo.conf
```

Flujo que sigue Ansible:

1. Lee el `.j2`.
2. Interpreta las expresiones Jinja2.
3. Sustituye las variables.
4. Genera el archivo final en el destino.

## Ejemplo real (nginx)

Plantilla:

```jinja
server {
    listen {{ puerto }};
    server_name {{ dominio }};
}
```

Variables:

```yaml
puerto: 443
dominio: ejemplo.com
```

Resultado:

```nginx
server {
    listen 443;
    server_name ejemplo.com;
}
```

## Relación con la programación

- Jinja2 **no** es un lenguaje de programación completo.
- Es un **DSL** (*Domain Specific Language*) orientado a generación de texto.
- Tiene: variables, control de flujo (`if`, `for`), filtros y expresiones.
- **No** está pensado para lógica compleja: no sustituye a Python.

## Cómo funciona por dentro

```
archivo.j2 → parser Jinja2 → AST (árbol sintáctico)
           → resolución de variables → evaluación de bloques → render final
```

## Las tres reglas de oro de la sintaxis

Escribes una plantilla con "huecos en blanco" y Ansible los rellena.

**1) Llaves dobles `{{ ... }}` — expresiones y variables**

Sirven para imprimir o inyectar un valor.

```jinja
Mi servidor se llama {{ ansible_facts['hostname'] }}
```

**2) Llave y porcentaje `{% ... %}` — lógica, bucles y condicionales**

Sirven para controlar el flujo: permiten usar `if` y `for`.

```jinja
{% if entorno == 'produccion' %}
ssl_protocols TLSv1.2 TLSv1.3;
{% endif %}
```

**3) Llave y almohadilla `{# ... #}` — comentarios**

Para dejar notas en la plantilla que no aparecen en el archivo final.

```jinja
{# nota: cambiar parámetro al migrar #}
```

## Uso nativo en Ansible

```yaml
- name: Generar archivo de configuración dinámico
  ansible.builtin.template:
    src: mi_configuracion.conf.j2      # plantilla con código Jinja2
    dest: /etc/mi_app/config.conf      # archivo final limpio que recibe el servidor
```

## Filtros: la navaja suiza de Jinja2

```
   valor  |  filtro  |  otro_filtro  →  resultado
     └──────► se encadenan de izquierda a derecha
```

| Filtro | Qué hace | Ejemplo |
|---|---|---|
| `default(x)` | valor por defecto si no existe | `{{ puerto \| default(80) }}` |
| `mandatory` | falla si no está definida | `{{ dominio \| mandatory }}` |
| `int` / `float` / `string` | convierte tipo | `{{ puerto \| int }}` |
| `upper` / `lower` / `capitalize` | texto | `{{ nombre \| upper }}` |
| `join(',')` | lista → texto | `{{ ips \| join(',') }}` |
| `split(',')` | texto → lista | `{{ csv \| split(',') }}` |
| `length` | tamaño | `{{ hosts \| length }}` |
| `unique` / `sort` / `reverse` | listas | `{{ puertos \| unique \| sort }}` |
| `select` / `reject` | filtra elementos | `{{ hosts \| select('match','web.*') \| list }}` |
| `map(attribute=...)` | extrae un campo | `{{ usuarios \| map(attribute='nombre') \| list }}` |
| `to_json` / `to_yaml` | serializa | `{{ config \| to_nice_yaml }}` |
| `from_json` | parsea | `{{ salida.stdout \| from_json }}` |
| `b64encode` / `b64decode` | base64 | `{{ texto \| b64encode }}` |
| `basename` / `dirname` | rutas | `{{ ruta \| basename }}` |
| `regex_replace` | sustitución | `{{ host \| regex_replace('^web','srv') }}` |
| `password_hash('sha512')` | hash de contraseña | para el módulo `user` |
| `ipaddr` | operaciones de red (colección `ansible.utils`) | `{{ red \| ipaddr('netmask') }}` |

## Bucles y condicionales en plantillas

```jinja
{% for site in nginx_sites %}
server {
    listen {{ site.port | default(80) }};
    server_name {{ site.domain }};
    {% if site.ssl | default(false) %}
    ssl_certificate /etc/ssl/{{ site.domain }}.crt;
    {% endif %}
}
{% endfor %}
```

## Espacios en blanco

```
{% for x in lista %}        ← deja una línea vacía por iteración
{%- for x in lista -%}      ← el guion recorta el espacio a ese lado
```

Muy útil para que los ficheros generados no salgan con líneas sueltas.

## Variables útiles dentro de una plantilla

| Variable | Contenido |
|---|---|
| `ansible_facts['hostname']` | nombre corto del host |
| `ansible_facts['fqdn']` | nombre completo |
| `ansible_facts['default_ipv4']['address']` | IP principal |
| `inventory_hostname` | cómo se llama el host en el inventario |
| `groups['webservers']` | lista de hosts de un grupo |
| `hostvars['web01']['variable']` | variable de otro host |
| `ansible_managed` | cabecera "fichero gestionado por Ansible" |

Cabecera recomendada en toda plantilla:

```jinja
# {{ ansible_managed }}
# NO editar a mano: lo sobreescribe Ansible
```

## Depurar plantillas

```bash
# ver el resultado sin aplicarlo
ansible-playbook site.yml --check --diff --tags plantillas

# evaluar una expresión suelta
ansible web01 -m debug -a "msg={{ hostvars['web01']['ansible_facts']['fqdn'] }}"
```

## Errores típicos

| Síntoma | Causa habitual |
|---|---|
| `'dict object' has no attribute 'x'` | la variable existe pero no esa clave: usa `default({})` |
| Comillas raras en el fichero final | mezclar `{{ }}` con comillas YAML: entrecomilla toda la expresión |
| El fichero sale con líneas en blanco | faltan los guiones `{%- -%}` |
| Rutas Windows rotas | `\` interpretado como escape: usa comillas simples |
