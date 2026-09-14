# Molecule

Framework de testing para roles y colecciones de Ansible.

## Qué hace

Levanta máquinas de usar y tirar (normalmente contenedores), aplica el rol, comprueba que el
resultado es el esperado y lo destruye todo.

```
   create ──► prepare ──► converge ──► idempotence ──► verify ──► destroy
     │           │           │              │              │          │
   levanta    requisitos   aplica el     lo aplica otra   comprueba   borra las
   instancias previos      rol           vez: debe dar    el estado   instancias
   (Docker…)  (repos,…)                  0 cambios        final
```

## Instalación

```bash
python -m venv .venv && source .venv/bin/activate
pip install ansible-core molecule "molecule-plugins[docker]"
```

> Los drivers han cambiado de paquete entre versiones (antes `molecule-docker`, después
> `molecule-plugins[docker]`) y las versiones recientes han simplificado el sistema de
> drivers. Revisa la documentación de la versión que instales.

## Estructura

```bash
ansible-galaxy role init nginx       # crear el rol
cd nginx
molecule init scenario               # crea molecule/default/
```

```
nginx/
├── defaults/  handlers/  tasks/  templates/  meta/
└── molecule/
    └── default/                 ← escenario (puede haber varios: default, windows, cluster…)
        ├── molecule.yml         ← plataformas, driver, provisioner
        ├── converge.yml         ← playbook que aplica el rol
        ├── prepare.yml          ← (opcional) preparar la instancia antes
        └── verify.yml           ← comprobaciones
```

### molecule.yml

```yaml
driver:
  name: docker

platforms:
  - name: debian12
    image: geerlingguy/docker-debian12-ansible:latest
    pre_build_image: true
    privileged: true                 # necesario para systemd dentro del contenedor
    cgroupns_mode: host
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    command: ""                      # usa el CMD de la imagen (arranca systemd)

  - name: rocky9
    image: geerlingguy/docker-rockylinux9-ansible:latest
    pre_build_image: true
    privileged: true
    cgroupns_mode: host
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    command: ""

provisioner:
  name: ansible
  inventory:
    group_vars:
      all:
        nginx_puerto: 8080

verifier:
  name: ansible
```

> Las imágenes normales de Debian o Rocky **no tienen systemd** ni Python: los módulos
> `service`/`systemd` fallan. Las imágenes de geerlingguy están preparadas justo para esto.

### converge.yml

```yaml
- name: Converge
  hosts: all
  become: true
  roles:
    - role: "{{ lookup('env', 'MOLECULE_PROJECT_DIRECTORY') | basename }}"
```

### verify.yml

```yaml
- name: Verify
  hosts: all
  become: true
  gather_facts: false
  tasks:
    - name: Recoger el estado de los servicios
      ansible.builtin.service_facts:

    - name: nginx debe estar arrancado
      ansible.builtin.assert:
        that:
          - ansible_facts.services['nginx.service'].state == 'running'
        fail_msg: "nginx no está corriendo"

    - name: Debe responder en el puerto configurado
      ansible.builtin.uri:
        url: "http://localhost:8080"
        status_code: 200
```

> `verify` comprueba **el resultado** (el servicio responde), no que las tareas se
> ejecutaron: eso ya lo dice `converge`.

## Comandos útiles

Ejecutar el `converge` con salida detallada (todo lo que va después de `--` se pasa a
`ansible-playbook`):

```bash
molecule converge -- -v
```

Ejecutar en modo comprobación (*dry run*), sin aplicar cambios:

```bash
molecule converge -- --check
```

| Comando | Qué hace |
|---|---|
| `molecule test` | ciclo completo: destruye, crea, aplica, idempotencia, verifica y destruye |
| `molecule create` | solo levanta las instancias |
| `molecule converge` | aplica el rol (repetible mientras desarrollas) |
| `molecule login -h debian12` | shell dentro de una instancia para investigar |
| `molecule idempotence` | segunda pasada; falla si alguna tarea da `changed` |
| `molecule verify` | lanza `verify.yml` |
| `molecule destroy` | borra las instancias |
| `molecule list` | instancias y su estado |
| `molecule reset` | limpia el estado de Molecule si se ha quedado incoherente |
| `molecule test --destroy=never` | ciclo completo pero deja las instancias vivas para depurar |
| `molecule test -s windows` | usar otro escenario |

### Ciclo de desarrollo

```
   molecule create
        │
        ▼
   ┌─► editar el rol ──► molecule converge ──► ¿funciona? ──no──┐
   │                                              │              │
   └──────────────────────────────────────────────┼──────────────┘
                                                  sí
                                                  ▼
                         molecule idempotence + molecule verify
                                                  ▼
                           molecule test  (ciclo limpio antes del MR)
```

`molecule test` completo tarda: mientras desarrollas, `converge` una y otra vez sobre la misma
instancia es mucho más rápido.

### Secuencia de `molecule test`

```
dependency → cleanup → destroy → syntax → create → prepare → converge
           → idempotence → side_effect → verify → cleanup → destroy
```

## Notas

- `-- ` separa los argumentos de Molecule de los que se reenvían a `ansible-playbook`.
- Aumentar la verbosidad (`-v`, `-vv`, `-vvv`) es lo primero que uso cuando una tarea falla
  sin mensaje claro.
- `--check` es útil para validar idempotencia antes de lanzar el `converge` real.

> Matiz sobre `--check`: comprueba qué **cambiaría**, pero no garantiza la idempotencia (las
> tareas `command`/`shell` se saltan en check). La prueba de idempotencia de verdad es
> `molecule idempotence`: aplicar el rol **dos veces** y exigir **0 cambios** en la segunda.

## Idempotencia: por qué falla y cómo arreglarlo

```
   PLAY RECAP (segunda pasada)
   debian12 : ok=12  changed=1  …        ← molecule idempotence falla

   CRITICAL Idempotence test failed because of the following tasks:
   *  => nginx : Recargar configuración
```

| Causa | Arreglo |
|---|---|
| `command` / `shell` siempre dan `changed` | `changed_when: false` (lecturas) o `creates:` / `removes:` |
| Plantilla con fecha u hora dentro | quitar el dato variable del fichero generado |
| `lineinfile` con una regex que no casa con la línea que escribe | ajustar `regexp` para que encuentre la línea ya modificada |
| Descargar siempre la "latest" | fijar versión o comprobar antes |

Ver [YAML, facts e idempotencia](../ansible/08-yaml-e-idempotencia.md).

## En CI (GitLab)

```yaml
molecule:
  stage: test
  image: python:3.12
  services:
    - docker:dind
  variables:
    DOCKER_HOST: tcp://docker:2375     # red interna del job de CI; nunca así fuera de CI
    DOCKER_TLS_CERTDIR: ""
  before_script:
    - pip install ansible-core molecule "molecule-plugins[docker]" docker
  script:
    - molecule test
```

> `docker:dind` requiere un runner con modo **privilegiado**. Si el runner no lo permite,
> alternativas: Podman como driver, o un runner con acceso al socket de Docker del host.

## ¿Y los roles de Windows?

Los contenedores no sirven para Windows Server. Opciones:

- Driver **delegated** (o "default") apuntando a una VM existente, con `create.yml`/`destroy.yml`
  propios (por ejemplo, restaurar un snapshot de la VM).
- Vagrant con una box de Windows en local.

## Errores típicos

| Error | Causa habitual |
|---|---|
| `Cannot connect to the Docker daemon` | Docker parado, o tu usuario no está en el grupo `docker` (ver [permisos](../linux-shell/04-permisos.md)) |
| `System has not been booted with systemd` | imagen sin systemd o faltan `privileged`/`cgroupns_mode`/volumen de cgroups |
| `the role 'nginx' was not found` | el nombre del rol no coincide con la carpeta: usar el `lookup` de `MOLECULE_PROJECT_DIRECTORY` o `role_name` en `meta/main.yml` |
| `/usr/bin/python: not found` | la imagen no trae Python: usar imágenes preparadas para Ansible |
| `Idempotence test failed` | ver la tabla de idempotencia |
| Estado raro tras un fallo a medias | `molecule destroy` o `molecule reset` |
