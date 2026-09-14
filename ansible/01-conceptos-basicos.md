# Ansible — Conceptos básicos

## Cómo funciona: arquitectura sin agentes

```
   ┌──────────────── NODO DE CONTROL ────────────────┐
   │  ansible.cfg   inventory   playbooks   roles    │
   │  colecciones   Python                           │
   └───────┬─────────────────────┬───────────────────┘
           │ SSH                 │ WinRM
           ▼                     ▼
   ┌──────────────┐      ┌──────────────────┐
   │ host Linux   │      │ host Windows     │   ← sin agente instalado
   │ (python)     │      │ (PowerShell)     │
   └──────────────┘      └──────────────────┘
```

Ansible **empuja** el módulo al host destino, lo ejecuta allí y recoge el resultado JSON.
No hay demonio corriendo en los nodos gestionados: solo hace falta conectividad y un
intérprete (Python en Linux, PowerShell en Windows).

## Configuración

- `ansible.cfg` es un archivo tipo INI (**no** XML, y tampoco YAML como el resto de Ansible).
- La configuración de la posición más baja en la jerarquía es la que acaba afectando a todo
  (gana la más cercana al proyecto).

### Orden de búsqueda del ansible.cfg

```
1. ANSIBLE_CONFIG (variable de entorno)   ← gana
2. ./ansible.cfg  (directorio actual)
3. ~/.ansible.cfg
4. /etc/ansible/ansible.cfg               ← pierde
```

En cuanto encuentra uno, **usa solo ese**: no se fusionan.

```ini
[defaults]
inventory = ./inventory
roles_path = ./roles
host_key_checking = False
forks = 20
stdout_callback = yaml

[galaxy]
server_list = mi_galaxy

[privilege_escalation]
become = True
become_method = sudo
```

## YAML: listas y diccionarios

- Si los elementos van precedidos de guion `-` → es una **lista**.
- Si cada elemento es `clave: valor` → es un **diccionario**.
- Python, al hacer merge, junta las listas dentro de un diccionario.

```yaml
# lista
paquetes:
  - nginx
  - git

# diccionario
servidor:
  nombre: web01
  puerto: 443
```

## Estructura de un proyecto

```
proyecto/
├── ansible.cfg
├── inventory/
│   ├── produccion.ini
│   └── group_vars/
│       ├── all.yml
│       └── webservers.yml
├── site.yml                 ← playbook principal
└── roles/
    └── nginx/
        ├── defaults/main.yml     valores configurables
        ├── vars/main.yml         valores internos
        ├── tasks/main.yml        las tareas
        ├── handlers/main.yml     acciones disparadas por notify
        ├── templates/nginx.conf.j2
        ├── files/
        └── meta/main.yml         dependencias del rol
```

## Jerarquía de ejecución

```
playbook  (lista de plays)
└── play        → a qué hosts y con qué permisos
    ├── pre_tasks
    ├── roles
    │   └── tasks    → cada tarea es un diccionario
    ├── tasks
    ├── post_tasks
    └── handlers     → se ejecutan al final si algo hizo notify
```

- Un **playbook** es una lista (de plays).
- Una **tarea (task)** es un diccionario.
- Un **rol** es un grupo de tareas con un carácter común.
- Las tareas se evalúan una a una, aunque estén agrupadas en bloques (`block`).

## FQCN

**FQCN** = *Fully Qualified Collection Name*.

```
namespace . coleccion . rol_o_modulo
    │          │            │
    │          │            └── nombre del módulo o rol
    │          └── nombre de la colección
    └── normalmente la organización
```

Ejemplo: `ansible.builtin.copy`, `ansible.windows.win_service`, `community.general.ufw`.

> Usar siempre el FQCN evita colisiones cuando dos colecciones tienen un módulo con el mismo
> nombre corto, y es obligatorio en las reglas modernas de `ansible-lint`.

## import vs include

| Directiva | Momento | Comportamiento |
|---|---|---|
| `import_*` | **Estático** | Se resuelve antes de ejecutar el playbook |
| `include_*` | **Dinámico** | Se resuelve en tiempo de ejecución |

```
   import_tasks              include_tasks
   ───────────               ─────────────
   se expande al parsear     se decide al ejecutar
   acepta tags heredados     puede usar variables en la ruta
   no admite loop            admite loop y when dinámico
```

## Handlers y notify

```
   tarea cambia algo (changed) ──notify──► handler
                                             │
                                    se ejecuta UNA vez
                                    al final del play
```

```yaml
tasks:
  - name: Copiar configuración
    ansible.builtin.template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Reiniciar nginx

handlers:
  - name: Reiniciar nginx
    ansible.builtin.service:
      name: nginx
      state: restarted
```

## Ejecución de scripts (contexto Linux)

- Tres formas típicas de ejecutar un script: `bash script.sh`, `zsh script.sh`,
  `fish script.fish`.
- `source script.sh` ejecuta el script **en la propia terminal actual** (no abre subshell),
  así que los cambios de entorno persisten.
- Un proceso hijo hereda todas las variables de entorno exportadas del padre.

## Comandos del día a día

```bash
ansible-playbook site.yml -i inventory/produccion.ini
ansible-playbook site.yml --check --diff       # dry run mostrando cambios
ansible-playbook site.yml --limit web01        # solo un host
ansible-playbook site.yml --tags nginx
ansible-playbook site.yml --start-at-task "Copiar configuración"
ansible-playbook site.yml -vvv                 # verbosidad para depurar
ansible all -m ping                            # comprobar conectividad
ansible web01 -m setup                         # ver todos los facts
ansible-inventory --graph                      # ver estructura del inventario
ansible-lint                                   # buenas prácticas
```
