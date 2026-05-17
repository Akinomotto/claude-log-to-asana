# Instrucción para Cowork Global Instructions

Pegá este bloque **una sola vez** en `Settings → Cowork → Edit Global Instructions`. A partir de ese momento se aplica automáticamente a todas las scheduled tasks de tu workspace, presentes y futuras.

---

**Al finalizar cualquier scheduled task de este workspace, invocá el skill `log-to-asana` en modo `--auto` pasándole como contexto el nombre de la scheduled task. La skill detectará si hubo entregables productivos (archivos creados, páginas Notion, eventos de Calendar, workflows n8n, etc.) y registrará el trabajo en Asana automáticamente en el proyecto correspondiente. Si no hubo entregables (la sesión fue solo lectura/búsqueda/conversación), no hará nada.**

---

## Cómo funciona

- La skill analiza qué tools de tipo "productivo" usaste durante la sesión (creación de páginas Notion, workflows n8n, eventos de Calendar, archivos en Drive, commits, etc.).
- Si detecta al menos un entregable, crea automáticamente una tarea en Asana en el proyecto que corresponda según las reglas de ruteo (`~/.claude/asana-project-mapping.json`).
- Si no hay match de proyecto, va al fallback "Inbox Claude".
- Si todo es lectura/búsqueda sin output, no crea nada.

## Scope más acotado

Si preferís activarlo solo en algunos Projects internos de Cowork (no en todo el workspace), pegá el mismo bloque en las **Project Instructions** de los proyectos elegidos en vez de Global Instructions.

## Cómo desactivarlo

Borrá el bloque de Global Instructions (o de las Project Instructions correspondientes). No hace falta tocar nada más.
