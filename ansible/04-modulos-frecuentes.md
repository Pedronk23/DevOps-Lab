# Ansible — Módulos frecuentes

## Mapa mental por categoría

```
   FICHEROS          PAQUETES/SERVICIOS      VARIABLES/FLUJO
   copy              package / apt / dnf     set_fact
   template          service / systemd       include_tasks
   file              win_package             include_vars
   lineinfile        win_service             debug
   blockinfile       win_feature             assert
   fetch                                     pause
   unarchive         COMANDOS                fail
   stat              command  (sin shell)
                     shell    (con pipes)    RED/ESPERAS
   USUARIOS          raw      (sin python)   uri
   user              script                  get_url
   group                                     wait_for
```

## `ansible.builtin`

### `set_fact`

Crea variables internas en tiempo de ejecución. Patrón habitual: construir un único objeto
con todos los valores centralizados y usarlo después en las tareas.

```yaml
- name: Centralizar configuración del sitio web
  ansible.builtin.set_fact:
    web_vars:
      site_name: "{{ site_name }}"
      port: 443
      path: 'C:\inetpub\wwwroot'
```

- Persiste durante todo el play para ese host.
- Con `cacheable: true` se guarda también en la caché de facts.
- Tiene prioridad alta: cuidado, pisa a `defaults` y a `group_vars`.

### `include_tasks`

Importa tareas desde otro archivo **en tiempo de ejecución**. Útil para instanciar
"features" o bloques repetidos con distintos parámetros.

```yaml
- name: Instalar cada feature
  ansible.builtin.include_tasks: tasks/feature.yml
  loop: "{{ features }}"
  loop_control:
    loop_var: feature
```

### `template` vs `copy`

```
  copy      →  fichero tal cual (estático)
  template  →  pasa por Jinja2 antes de escribir (dinámico)
```

```yaml
- name: Config dinámica
  ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/app/app.conf
    owner: root
    group: root
    mode: "0640"
    backup: true            # guarda copia del anterior
    validate: "nginx -t -c %s"   # valida antes de dejarlo en su sitio
  notify: Reiniciar app
```

### `lineinfile` y `blockinfile`

Para retocar ficheros que no controlas por completo.

```yaml
- name: Desactivar login de root por SSH
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PermitRootLogin'
    line: 'PermitRootLogin no'
    validate: '/usr/sbin/sshd -t -f %s'
```

> Si vas a tocar más de dos líneas, mejor un `template` del fichero completo.

### `command` vs `shell` vs `raw`

| Módulo | Usa shell | Cuándo |
|---|---|---|
| `command` | no | por defecto: más seguro, sin pipes ni redirecciones |
| `shell` | sí | cuando necesitas `\|`, `>`, `&&`, variables de entorno |
| `raw` | sí, sin Python | bootstrap de un host que aún no tiene Python |

```yaml
- name: Comando idempotente
  ansible.builtin.command: /opt/instalar.sh
  args:
    creates: /opt/instalado.flag      # si existe, no lo ejecuta
  changed_when: false                 # o controla tú cuándo es "changed"
```

### `uri` y `get_url`

```yaml
- name: Esperar a que la API responda
  ansible.builtin.uri:
    url: https://app.ejemplo.com/health
    status_code: 200
  register: health
  retries: 10
  delay: 5
  until: health.status == 200
```

### `assert` y `fail`

Validar precondiciones antes de romper nada.

```yaml
- name: Validar variables obligatorias
  ansible.builtin.assert:
    that:
      - dominio is defined
      - nginx_port | int > 0
    fail_msg: "Falta 'dominio' o el puerto no es válido"
```

## `ansible.windows`

### `win_service`

Gestiona servicios de Windows (estado, arranque, cuenta de ejecución).

```yaml
- name: Asegurar servicio arrancado
  ansible.windows.win_service:
    name: MiServicio
    state: started
    start_mode: auto
```

### Otros muy usados

```yaml
- name: Instalar feature de IIS
  ansible.windows.win_feature:
    name: Web-Server
    include_management_tools: true

- name: Crear carpeta
  ansible.windows.win_file:
    path: C:\apps\MiApp
    state: directory

- name: Clave de registro
  ansible.windows.win_regedit:
    path: HKLM:\SOFTWARE\MiApp
    name: Version
    data: "1.2.0"
    type: string
```

## Cómo consultar la documentación sin salir del terminal

```bash
ansible-doc ansible.builtin.template        # documentación completa
ansible-doc -s ansible.windows.win_service  # snippet listo para pegar
ansible-doc -l | grep win_                  # listar módulos
```
