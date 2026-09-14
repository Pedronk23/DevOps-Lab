# AWX (Automation Controller)

Proyecto open source de Red Hat. Básicamente es una **interfaz web + API REST** que se pone por
encima de `ansible-playbook` para gestionar la automatización de forma centralizada.

## Qué aporta

| Capacidad | Detalle |
|---|---|
| **UI y API** | lanzar playbooks desde la web, o vía API/webhooks desde el CI |
| **Job Templates** | encapsulan playbook + inventario + credenciales + variables extra, con formularios (*surveys*) para parametrizar |
| **Gestión de credenciales** | las guarda cifradas y las inyecta en tiempo de ejecución. Soporta backends externos (Vault, OpenBao) |
| **Inventarios dinámicos** | se sincronizan desde fuentes externas: una API, un CMDB, un cloud provider |
| **RBAC** | organizaciones, equipos y roles, para que cada equipo solo vea y ejecute lo suyo |
| **Execution Environments** | imágenes de contenedor con la versión de ansible-core, colecciones y dependencias Python que necesita cada job. Sustituyeron a los viejos virtualenvs |
| **Automation Mesh** | (más en AAP que en AWX puro): nodos de ejecución distribuidos para llegar a redes segmentadas |
| **Trazabilidad** | histórico de ejecuciones, output completo, quién lanzó qué y cuándo, notificaciones |

## Arquitectura resumida

- **PostgreSQL** como base de datos.
- **Redis** para la cola de mensajes.
- Un componente **web** (Django).
- **Nodos de ejecución** que levantan contenedores efímeros por cada job.

## Flujo de un job

```
   usuario / webhook del CI
            │
            ▼
   ┌──────────────────┐
   │  Job Template    │  playbook + inventario + credenciales + survey
   └────────┬─────────┘
            ▼
   ┌──────────────────┐      levanta un contenedor efímero
   │  Execution Node  │───►  con el Execution Environment
   └────────┬─────────┘
            ▼
   ┌──────────────────┐
   │ ansible-playbook │───► hosts del inventario (SSH / WinRM)
   └────────┬─────────┘
            ▼
   logs + estado → PostgreSQL → visibles en la UI y la API
```

## Jerarquía de objetos

```
   Organización
   ├── Equipos (usuarios y roles)
   ├── Proyectos        → de dónde sale el código (Git)
   ├── Inventarios      → hosts y grupos (estáticos o dinámicos)
   ├── Credenciales     → machine, vault, registry, cloud
   ├── Execution Environments
   ├── Job Templates    → la unidad que se lanza
   └── Workflow Templates → encadenan varios job templates con ramas
```

## Workflows

```
        ┌──────────────┐   éxito   ┌──────────────┐
        │ desplegar    │──────────►│ smoke tests  │
        └──────┬───────┘           └──────┬───────┘
               │ fallo                    │ fallo
               ▼                          ▼
        ┌──────────────┐           ┌──────────────┐
        │ rollback     │           │ notificar    │
        └──────────────┘           └──────────────┘
```

Permiten condicionar el siguiente paso a *success*, *failure* o *always*.

## Lanzar jobs desde el CI

```bash
curl -k -X POST \
  -H "Authorization: Bearer $AWX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"extra_vars": {"entorno": "produccion"}}' \
  https://awx.interno.local/api/v2/job_templates/42/launch/
```

Consultar el resultado:

```bash
curl -k -H "Authorization: Bearer $AWX_TOKEN" \
  https://awx.interno.local/api/v2/jobs/1234/ | jq .status
```

## AWX vs AAP (Ansible Automation Platform)

| | AWX | AAP |
|---|---|---|
| Licencia | open source, comunidad | producto de Red Hat con soporte |
| Ritmo de releases | rápido, sin garantías | versiones estables certificadas |
| Automation Mesh | limitado | completo |
| Hub de colecciones certificadas | no | sí |
| Uso típico | laboratorio, empresas que asumen el mantenimiento | producción con soporte |

## Buenas prácticas

- El playbook vive en Git; AWX solo lo ejecuta (proyecto sincronizado, nunca editar allí).
- Credenciales siempre en AWX o en Vault/OpenBao, nunca en el repo.
- Surveys para parametrizar en vez de duplicar job templates.
- Un Execution Environment por stack de dependencias, versionado como cualquier imagen.
- RBAC por equipo: cada uno solo ve sus inventarios y plantillas.
