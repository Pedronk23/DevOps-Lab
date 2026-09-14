# Ansible — Directivas y conceptos clave

## Piezas básicas

| Concepto | Qué es |
|---|---|
| **Playbook** | archivo YAML que describe las tareas que quieres automatizar en tus servidores. Define qué hacer, dónde y cómo |
| **Módulos** | las herramientas de trabajo real que ejecuta Ansible tras bambalinas (instalar con apt, copiar, gestionar servicios). Los playbooks los orquestan |
| **Roles** | la mejor práctica para organizar y modularizar el código. En lugar de tener un único archivo YAML gigantesco |

## Inventarios dinámicos

Utilizan un script o plugin (en Python, Bash…) para consultar una API externa en tiempo real.

En AWS, Google Cloud o Kubernetes: busca automáticamente las instancias activas y sus IPs según
sus etiquetas.

## register y when

Combinar estas dos directivas permite crear lógica.

- **register**: guarda el resultado de la ejecución dentro de una variable.
- **when**: aplica una condición para ejecutar una tarea solo si se cumple.

```yaml
- name: Comprobar si existe el fichero
  ansible.builtin.stat:
    path: /etc/mi_app/config.conf
  register: config_file

- name: Crear config solo si no existe
  ansible.builtin.template:
    src: config.conf.j2
    dest: /etc/mi_app/config.conf
  when: not config_file.stat.exists
```

## Bucles (loop)

Permiten repetir una misma tarea iterando sobre una lista de elementos sin necesidad de
duplicar código.

```yaml
- name: Instalar paquetes
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - git
    - curl
```

## Tareas asíncronas

Ansible normalmente ejecuta las tareas de forma **síncrona**: espera que una termine para
empezar otra. Las asíncronas se usan en procesos pesados.

- **async**: define el tiempo máximo que se le permite ejecutar a la tarea.
- **poll**: define cada cuántos segundos Ansible preguntará si la tarea ha terminado.
  Con `poll: 0` dispara la tarea y continúa inmediatamente con el playbook.

```yaml
- name: Proceso largo
  ansible.builtin.command: /opt/script_pesado.sh
  async: 3600
  poll: 0
```

## Magic variables

Variables reservadas que Ansible crea y actualiza automáticamente durante la ejecución.

| Variable | Qué contiene |
|---|---|
| `inventory_hostname` | el host actual, según está escrito en el archivo de inventario |
| `hostvars` | permite acceder a las variables o facts de otro servidor dentro del mismo playbook |

## Blocks (bloques)

Permiten agrupar varias tareas relacionadas para aplicarles condicionales (`when`), directivas
de ejecución, o para gestionar errores al estilo `try/catch/finally` de la programación.

| Sección | Cuándo se ejecuta |
|---|---|
| `block` | tareas principales a ejecutar |
| `rescue` | tareas que solo se ejecutan si alguna dentro del block falla |
| `always` | se ejecutan siempre, falle o no el bloque |

```yaml
- block:
    - name: Intentar despliegue
      ansible.builtin.command: /opt/deploy.sh
  rescue:
    - name: Revertir
      ansible.builtin.command: /opt/rollback.sh
  always:
    - name: Limpiar temporales
      ansible.builtin.file:
        path: /tmp/deploy
        state: absent
```

## Ansible Vault

Sistema de encriptación integrado en Ansible para proteger datos sensibles.

```bash
ansible-vault encrypt vars/secretos.yml
ansible-vault view vars/secretos.yml
ansible-playbook site.yml --ask-vault-pass
```

## Estrategias de ejecución y paralelismo

```
   linear (por defecto)              free
   ────────────────────              ────────────────────
   tarea1: web01 web02 web03         web01: t1 t2 t3 t4
           ↓ (espera a todos)        web02: t1 t2 (más lento, no bloquea)
   tarea2: web01 web02 web03         web03: t1 t2 t3
```

```yaml
- hosts: webservers
  strategy: linear      # linear | free | host_pinned
  serial: 2             # despliegue por lotes de 2 hosts (canary/rolling)
  max_fail_percentage: 20
```

`forks` (en `ansible.cfg`) controla cuántos hosts se atacan en paralelo; `serial` controla
en cuántos lotes se divide el play. Combinados dan un despliegue rolling controlado:

```
   serial: 1  →  web01 ──► comprobar ──► web02 ──► comprobar ──► web03
```

## Delegación y ejecución única

```yaml
- name: Sacar el host del balanceador
  ansible.builtin.uri:
    url: "http://lb.interno/disable/{{ inventory_hostname }}"
  delegate_to: lb.interno          # se ejecuta en otro host

- name: Crear la BBDD una sola vez
  ansible.builtin.command: /opt/crear_db.sh
  run_once: true                   # solo en el primer host del play

- name: Tarea local
  ansible.builtin.command: git rev-parse HEAD
  delegate_to: localhost
  become: false
```

## Gestión de errores

```yaml
- name: Puede fallar sin parar el play
  ansible.builtin.command: /opt/opcional.sh
  ignore_errors: true

- name: Controlar qué se considera fallo
  ansible.builtin.command: /opt/check.sh
  register: res
  failed_when: res.rc not in [0, 2]
  changed_when: res.rc == 0
```

```
   ¿falla una tarea?
        │
        ├── ignore_errors: true      → sigue
        ├── dentro de block+rescue   → ejecuta rescue
        ├── any_errors_fatal: true   → aborta el play en TODOS los hosts
        └── por defecto              → ese host queda fuera del resto del play
```

## Tags

```yaml
tasks:
  - name: Instalar paquetes
    ansible.builtin.package:
      name: nginx
      state: present
    tags: [instalacion, nginx]
```

```bash
ansible-playbook site.yml --tags nginx
ansible-playbook site.yml --skip-tags instalacion
ansible-playbook site.yml --list-tags
```

Tags especiales: `always` (se ejecuta siempre) y `never` (solo si la pides explícitamente).

## Loops: variantes útiles

```yaml
# lista de diccionarios
- name: Crear usuarios
  ansible.builtin.user:
    name: "{{ item.nombre }}"
    groups: "{{ item.grupos }}"
  loop:
    - { nombre: ana, grupos: sudo }
    - { nombre: luis, grupos: docker }

# recorrer un diccionario
- name: Variables de entorno
  ansible.builtin.lineinfile:
    path: /etc/environment
    line: "{{ item.key }}={{ item.value }}"
  loop: "{{ entorno | dict2items }}"

# reintentos hasta que se cumpla una condición
- name: Esperar al servicio
  ansible.builtin.uri:
    url: http://localhost:8080/health
  register: r
  until: r.status == 200
  retries: 12
  delay: 5
```

## Contenido de una variable registrada

```
   register: res
   └── res.rc          código de salida
       res.stdout      salida estándar
       res.stdout_lines lista de líneas
       res.stderr      errores
       res.changed     ¿cambió algo?
       res.failed      ¿falló?
       res.results[]   una entrada por iteración si había loop
```

## Ansible Vault: comandos completos

```bash
ansible-vault create secretos.yml
ansible-vault edit secretos.yml
ansible-vault view secretos.yml
ansible-vault encrypt vars/produccion.yml
ansible-vault decrypt vars/produccion.yml
ansible-vault rekey secretos.yml                  # cambiar la contraseña
ansible-vault encrypt_string 'p4ssw0rd' --name 'db_password'   # cifrar un solo valor

ansible-playbook site.yml --vault-password-file ~/.vault_pass
ansible-playbook site.yml --ask-vault-pass
```

```
   vault-id: varias contraseñas para distintos entornos
   ansible-vault encrypt --vault-id prod@prompt vars/prod.yml
   ansible-playbook site.yml --vault-id prod@~/.vault_prod
```

## Chuleta de diagnóstico

| Problema | Qué mirar |
|---|---|
| "changed" en cada ejecución | falta `creates` / `changed_when` |
| Variable con valor inesperado | precedencia: ver [vars vs defaults](03-variables-vars-defaults.md) |
| Falla solo en algunos hosts | `--limit` + `-vvv` en ese host |
| Lento | `gather_facts: false`, subir `forks`, activar pipelining |
| No conecta | `ansible host -m ping`, revisar `ansible_user` y las claves |
