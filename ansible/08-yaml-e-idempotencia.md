# Ansible — YAML, gather facts e idempotencia

## YAML

Guarda datos estructurados de una forma que sea muy fácil de leer para humanos y máquinas.
Se usa muchísimo en programación y DevOps para escribir archivos de configuración.

A diferencia de JSON o XML, **no usa llaves `{}` ni etiquetas `<tags>`**. En su lugar, utiliza
**sangrías (espacios)** para organizar la información en listas o pares clave-valor.

### Listas y diccionarios

**Lista**: secuencia ordenada de elementos. Cada elemento tiene una posición fija llamada
**índice** (empieza en 0). Se busca por posición.

```yaml
frutas:
  - manzana
  - platano
  - pera
```

**Diccionario**: estructura clave-valor. Aquí importa la etiqueta: se busca por su nombre.

```yaml
usuario:
  nombre: Pedro
  edad: 28
  ciudad: Sevilla
```

## Gather facts

Proceso automático que ocurre al inicio de un playbook: se conecta al servidor remoto y
averigua todo sobre él antes de ejecutar cualquier tarea (SO, IPs, memoria, discos…).

```yaml
- hosts: all
  gather_facts: true    # por defecto está activo
```

## Formatos

El 99 % de los archivos de Ansible son YAML; para los archivos de inventario también se usa
`.ini`.

## Idempotencia

Puedes ejecutar el playbook 100 veces y el resultado será siempre el mismo, sin romper nada en
el camino.

> Si le dices a Ansible "instala Apache", la primera vez lo instala; la segunda mira el
> servidor, ve que Apache ya está y no hace nada. Esto evita los típicos problemas de un
> script de Bash, que repetiría la acción a ciegas.

## Trampas clásicas de YAML

| Escribes | Ansible entiende | Solución |
|---|---|---|
| `version: 1.10` | `1.1` (float) | `version: "1.10"` |
| `enabled: yes` | `true` (booleano) | comillas si querías el texto "yes" |
| `puerto: 08080` | error de octal | `puerto: 8080` |
| `password: *secreto` | referencia a un anchor | entrecomillar |
| Tabulador para indentar | error de parseo | **solo espacios** (2 por nivel) |
| `mode: 644` | 644 decimal, no permisos | `mode: "0644"` |

## Bloques de texto multilínea

```yaml
# | conserva los saltos de línea
script: |
  #!/bin/bash
  echo "linea 1"
  echo "linea 2"

# > los convierte en espacios (una sola línea larga)
descripcion: >
  Este texto acabará
  en una sola línea.
```

## Anchors y alias (reutilizar bloques)

```yaml
defaults: &comunes
  owner: root
  group: root
  mode: "0644"

ficheros:
  - <<: *comunes
    path: /etc/app/a.conf
  - <<: *comunes
    path: /etc/app/b.conf
```

## Facts: qué te da `gather_facts`

```
   ansible_facts
   ├── hostname / fqdn
   ├── distribution / distribution_version     (Ubuntu, 24.04)
   ├── os_family                               (Debian, RedHat, Windows)
   ├── default_ipv4.address / .gateway
   ├── processor_vcpus / memtotal_mb
   ├── mounts[] / devices{}
   ├── service_mgr                             (systemd)
   └── env{}                                   variables de entorno
```

```bash
ansible web01 -m setup                              # todos
ansible web01 -m setup -a "filter=ansible_distribution*"
```

Usarlos para escribir plays portables:

```yaml
- name: Instalar Apache según la distro
  ansible.builtin.package:
    name: "{{ 'httpd' if ansible_facts['os_family'] == 'RedHat' else 'apache2' }}"
    state: present
```

Si no los necesitas, desactívalos: el arranque del play es notablemente más rápido.

```yaml
- hosts: all
  gather_facts: false
```

## Idempotencia: los tres estados de una tarea

```
   ok        → ya estaba como se pide, no toca nada
   changed   → ha tenido que modificar algo
   failed    → no ha podido
```

```
   1ª ejecución:  changed=8   ok=3
   2ª ejecución:  changed=0   ok=11     ← objetivo: todo ok
```

> Si un playbook nunca llega a `changed=0` al repetirlo, hay tareas no idempotentes.
> Casi siempre son `command`/`shell` sin `creates`, `removes` o `changed_when`.

```yaml
- name: Generar informe (no cambia el sistema)
  ansible.builtin.command: /opt/informe.sh
  changed_when: false          # nunca reportará changed

- name: Ejecutar solo una vez
  ansible.builtin.command: /opt/instalar.sh
  args:
    creates: /opt/.instalado   # si el fichero existe, salta la tarea
```

## Modo comprobación (dry run)

```bash
ansible-playbook site.yml --check --diff
```

```
  --check  → simula, no aplica
  --diff   → muestra el antes/después de ficheros y plantillas
```

Tareas que no se pueden simular se marcan con:

```yaml
  check_mode: false     # esta tarea sí se ejecuta de verdad en --check
```
