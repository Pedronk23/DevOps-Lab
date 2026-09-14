# Ansible — Variables: `vars` vs `defaults`

## El problema: demasiados sitios donde definir variables

```
        MENOS prioridad
   ┌──────────────────────────────────┐
   │ role/defaults/main.yml           │  ← pensado para que lo pises
   ├──────────────────────────────────┤
   │ inventory group_vars/all         │
   ├──────────────────────────────────┤
   │ inventory group_vars/<grupo>     │
   ├──────────────────────────────────┤
   │ inventory host_vars/<host>       │
   ├──────────────────────────────────┤
   │ facts del host (gather_facts)    │
   ├──────────────────────────────────┤
   │ play vars / vars_files           │
   ├──────────────────────────────────┤
   │ role/vars/main.yml               │  ← valores internos del rol
   ├──────────────────────────────────┤
   │ block vars → task vars           │
   ├──────────────────────────────────┤
   │ include_vars                     │
   ├──────────────────────────────────┤
   │ set_fact / registered vars        │
   ├──────────────────────────────────┤
   │ extra vars  (-e)                 │  ← gana SIEMPRE
   └──────────────────────────────────┘
        MÁS prioridad
```

(La lista oficial tiene 22 niveles; estos son los que se usan en la práctica.)

## `defaults/` (dentro del rol)

- Valores **configurables**.
- Están pensados para que los sobreescriba el usuario del rol.
- Es la precedencia más baja: cualquier cosa los pisa.

```yaml
# roles/nginx/defaults/main.yml
nginx_port: 80
nginx_worker_processes: auto
nginx_sites: []
```

## `vars/` (dentro del rol)

- Valores **internos** del rol.
- No deberían cambiar entre ejecuciones ni entre entornos.
- Tienen **más prioridad** que los defaults (cuesta pisarlos: hacen falta extra vars).

```yaml
# roles/nginx/vars/main.yml
nginx_config_path: /etc/nginx/nginx.conf
nginx_service_name: nginx
```

## Regla práctica

> Si el que consume el rol puede querer cambiarlo → `defaults/`.
> Si es un detalle de implementación del rol → `vars/`.

## group_vars y host_vars

```
inventory/
├── produccion.ini
├── group_vars/
│   ├── all.yml           → todos los hosts
│   ├── webservers.yml    → el grupo webservers
│   └── webservers/       → también vale carpeta: vars.yml + vault.yml
└── host_vars/
    └── web01.yml         → solo ese host
```

Patrón habitual: separar lo público de lo cifrado.

```
group_vars/webservers/
├── vars.yml     → variables normales, referencian a las vault
└── vault.yml    → cifrado con ansible-vault
```

```yaml
# vars.yml
db_password: "{{ vault_db_password }}"
```

## Extra vars y prompts

```bash
ansible-playbook site.yml -e "entorno=produccion nginx_port=8080"
ansible-playbook site.yml -e @variables.yml
```

```yaml
vars_prompt:
  - name: entorno
    prompt: "¿Qué entorno?"
    private: false
```

## Valores por defecto en Jinja2

```yaml
# si la variable no existe, no falla
puerto: "{{ nginx_port | default(80) }}"

# obligar a que exista, con mensaje claro
dominio: "{{ dominio | mandatory }}"
```

## Ver qué valor está ganando

```bash
ansible web01 -m debug -a "var=nginx_port"
ansible-playbook site.yml -e "x=1" --check -vvv | grep nginx_port
```

> Truco de depuración: si una variable "no coge el valor", casi siempre es que está definida
> en un nivel de mayor prioridad (típicamente en `vars/` del rol o en un `set_fact`).
