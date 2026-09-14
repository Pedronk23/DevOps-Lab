# Ansible en Linux vs Windows

## Comparativa

| Aspecto | Linux | Windows |
|---|---|---|
| Conexión | SSH (puerto 22) | WinRM (5985 HTTP / 5986 HTTPS) o SSH |
| Módulos | genéricos (`copy`, `service`, …) | prefijo `win_` (`win_copy`, `win_service`, …) |
| Colección principal | `ansible.builtin` | `ansible.windows`, `community.windows` |
| Shell y scripts | bash / zsh | PowerShell |
| Intérprete necesario | Python | PowerShell 3.0+ y .NET 4.0+ |
| Elevación | `become: true` (sudo) | `become_method: runas` |
| Rutas | `/etc/nginx/nginx.conf` | `C:\inetpub\wwwroot\index.html` |
| Gestor de paquetes | apt / yum / dnf | `win_chocolatey`, `win_package` |

## Cómo viaja la tarea

```
                LINUX                                WINDOWS
   ┌────────────────────────────┐       ┌────────────────────────────────┐
   │ nodo de control            │       │ nodo de control                │
   │   genera módulo Python     │       │   genera script PowerShell     │
   └─────────────┬──────────────┘       └───────────────┬────────────────┘
                 │ SSH                                  │ WinRM (SOAP/HTTP)
                 ▼                                      ▼
   ┌────────────────────────────┐       ┌────────────────────────────────┐
   │ /tmp/ansible-tmp-.../      │       │ %TEMP%\ansible-tmp-...         │
   │ python3 modulo.py          │       │ powershell.exe -File modulo.ps1│
   │        ↓ JSON              │       │        ↓ JSON                  │
   └────────────────────────────┘       └────────────────────────────────┘
```

## Inventario

```ini
[webservers]
web01.ejemplo.local
web02.ejemplo.local

[webservers:vars]
ansible_connection=ssh
ansible_user=devops

[windows]
srv01.lab.local

[windows:vars]
ansible_connection=winrm
ansible_user=devops
ansible_winrm_transport=ntlm
ansible_winrm_server_cert_validation=ignore
ansible_port=5986
```

## Equivalencias de módulos

| Tarea | Linux | Windows |
|---|---|---|
| Copiar fichero | `ansible.builtin.copy` | `ansible.windows.win_copy` |
| Plantilla | `ansible.builtin.template` | `ansible.windows.win_template` |
| Servicio | `ansible.builtin.service` | `ansible.windows.win_service` |
| Paquete | `ansible.builtin.package` | `ansible.windows.win_package` / `win_chocolatey` |
| Comando | `ansible.builtin.command` | `ansible.windows.win_command` |
| Shell | `ansible.builtin.shell` | `ansible.windows.win_shell` (PowerShell) |
| Usuario | `ansible.builtin.user` | `ansible.windows.win_user` |
| Directorio/fichero | `ansible.builtin.file` | `ansible.windows.win_file` |
| Registro | — | `ansible.windows.win_regedit` |
| Feature del SO | — | `ansible.windows.win_feature` |
| Reiniciar | `ansible.builtin.reboot` | `ansible.windows.win_reboot` |

## Notas prácticas

- El módulo correcto depende del SO destino: en Windows casi siempre está en la colección
  `ansible.windows` (o `community.windows`) con prefijo `win_`.
- Las rutas Windows usan `\`, lo que obliga a vigilar el escapado en YAML y en Jinja2:
  usa **comillas simples** (`'C:\ruta\fichero'`) o dobles barras.
- WinRM tolera mejor el corte de conexión que SSH: si se pierde, se puede retomar la sesión
  después sin problema.
- `win_reboot` espera a que la máquina vuelva; no hace falta hacer trucos con `wait_for`.
- Windows no tiene idempotencia "gratis" en `win_shell`/`win_command`: usa `creates`,
  `removes` o un `when` para no repetir acciones.

## Ejemplo de play mixto

```yaml
- name: Configurar servidores Linux
  hosts: webservers
  become: true
  tasks:
    - name: Asegurar nginx instalado
      ansible.builtin.package:
        name: nginx
        state: present

- name: Configurar servidores Windows
  hosts: windows
  tasks:
    - name: Asegurar IIS instalado
      ansible.windows.win_feature:
        name: Web-Server
        include_management_tools: true
      register: iis

    - name: Reiniciar si hace falta
      ansible.windows.win_reboot:
      when: iis.reboot_required
```

## Comprobar conectividad

```bash
ansible webservers -m ping            # módulo ping (Linux)
ansible windows -m win_ping           # equivalente para Windows
```
