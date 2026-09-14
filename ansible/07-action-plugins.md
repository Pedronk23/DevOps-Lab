# Ansible — Action plugins

Los **plugins de acción** actúan junto con los módulos para ejecutar las acciones requeridas
por las tareas del playbook. Normalmente se ejecutan automáticamente en segundo plano,
realizando el trabajo previo antes de que los módulos se ejecuten.

## Activación

Puedes habilitar un action plugin personalizado:

- dejándolo en el directorio `action_plugins/` adyacente a tu playbook,
- dentro de un rol,
- o poniéndolo en una de las rutas de directorios de action plugins configuradas en
  `ansible.cfg` (`action_plugins`).

## Uso

Los plugins de acción se ejecutan **por defecto** cuando se utiliza un módulo asociado;
no se requiere ninguna acción adicional.

## Listado

No puedes listar action plugins directamente: aparecen como sus módulos equivalentes.

## Para qué sirven

**Preparan el terreno**: módulos como `copy`, `template` o `fetch` usan action plugins para
empaquetar archivos localmente, verificar si existen o calcular hashes antes de enviar algo
al nodo remoto.

- **Evitar conexiones innecesarias**: si una condición se puede resolver localmente, el action
  plugin puede decidir no conectarse al host.
- **Controlar el flujo**: permite realizar lógica compleja (bucles locales o condicionales)
  antes de invocar al módulo real en el destino.

## Dónde encaja en el flujo de una tarea

```
   PLAYBOOK
      │
      ▼
  ┌──────────────────────────────────────────┐
  │ ACTION PLUGIN (en el nodo de control)    │
  │  · lee ficheros locales                  │
  │  · renderiza plantillas                  │
  │  · calcula hashes / compara              │
  │  · decide si hace falta conectar         │
  └────────────────┬─────────────────────────┘
                   │ solo si hace falta
                   ▼
  ┌──────────────────────────────────────────┐
  │ MÓDULO (en el host destino)              │
  │  · escribe el fichero, arranca servicio… │
  │  · devuelve JSON                         │
  └──────────────────────────────────────────┘
```

## Ejemplos de módulos que dependen de un action plugin

| Módulo | Qué hace su action plugin antes |
|---|---|
| `copy` | lee el fichero local, calcula su checksum y solo transfiere si difiere |
| `template` | renderiza el Jinja2 **en el nodo de control** y luego lo copia |
| `fetch` | prepara el destino local antes de traerse el fichero |
| `assemble` | concatena fragmentos locales |
| `unarchive` | decide si descomprime en local o en remoto |

## Estructura mínima de un action plugin propio

```
roles/mi_rol/
└── action_plugins/
    └── mi_accion.py
```

```python
from ansible.plugins.action import ActionBase

class ActionModule(ActionBase):
    def run(self, tmp=None, task_vars=None):
        result = super(ActionModule, self).run(tmp, task_vars)
        # lógica en el nodo de control
        valor = self._task.args.get('valor')
        if not valor:
            return {'failed': True, 'msg': 'falta valor'}
        # delegar en un módulo si hace falta tocar el host
        return self._execute_module(
            module_name='ansible.builtin.debug',
            module_args={'msg': valor},
            task_vars=task_vars,
        )
```

## Tipos de plugin en Ansible (para situarlos)

| Tipo | Se ejecuta en | Para qué |
|---|---|---|
| **action** | nodo de control | preparar el terreno antes del módulo |
| **connection** | nodo de control | cómo se conecta (ssh, winrm, local, docker) |
| **callback** | nodo de control | formato de la salida, notificaciones |
| **lookup** | nodo de control | leer datos externos (`lookup('file', ...)`) |
| **filter** / **test** | nodo de control | funciones de Jinja2 |
| **inventory** | nodo de control | fuentes de inventario dinámico |
| **strategy** | nodo de control | orden de ejecución (linear, free, host_pinned) |
| **module** | host destino | el trabajo real |
