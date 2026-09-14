# Ansible

Notas de automatización con Ansible: sintaxis, estructura de proyectos, diferencias
Linux/Windows, plantillas, distribución de colecciones y ejecución centralizada.

## Orden de lectura recomendado

| # | Nota | Qué cubre |
|---|---|---|
| 01 | [Conceptos básicos](01-conceptos-basicos.md) | ansible.cfg, YAML, FQCN, import vs include |
| 02 | [Linux vs Windows](02-linux-vs-windows.md) | SSH/WinRM, módulos win_, rutas |
| 03 | [vars vs defaults](03-variables-vars-defaults.md) | precedencia de variables |
| 04 | [Módulos frecuentes](04-modulos-frecuentes.md) | set_fact, include_tasks, win_service |
| 05 | [Jinja2](05-jinja2.md) | plantillas, sintaxis, filtros |
| 06 | [Galaxy privado y colecciones](06-galaxy-y-colecciones.md) | publicar y consumir colecciones |
| 07 | [Action plugins](07-action-plugins.md) | qué pasa antes de que corra el módulo |
| 08 | [YAML, facts e idempotencia](08-yaml-e-idempotencia.md) | fundamentos del lenguaje |
| 09 | [Directivas y conceptos clave](09-directivas-y-conceptos.md) | register/when, loops, async, blocks |
| 10 | [AWX](10-awx.md) | ejecución centralizada y RBAC |
