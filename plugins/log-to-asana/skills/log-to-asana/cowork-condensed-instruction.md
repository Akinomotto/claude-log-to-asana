# Versión condensada para Cowork (Global o Project Instructions)

Cowork desktop no soporta marketplaces personalizados, así que el plugin no se puede instalar ahí. Esta versión condensada (~300 palabras) replica el comportamiento esencial de la skill embebiendo las instrucciones directamente en el prompt de Cowork.

**Dónde pegarla:**
- **Global Instructions** (`Settings → Cowork → Edit Global Instructions`): aplica a todas las sesiones y scheduled tasks del workspace. Más práctico, pero overhead de ~600 tokens por sesión.
- **Project Instructions** de proyectos específicos: solo en proyectos donde quieras tracking. Más targeted, sin overhead general.

**Lo que se mantiene** vs. el plugin completo en Code: detección de entregables, ruteo por typeahead con fallback a Inbox Claude, creación de tarea con campos clave, custom fields de tiempo, inferencia de completed/trunca.

**Lo que se pierde:** mapping local con 16 reglas personalizadas (Cowork hace typeahead cada vez), subtareas inteligentes, aprendizaje incremental de reglas, tag automático Claude-Code (el MCP no expone tags; se aplica manualmente o desde Code).

---

## Bloque para pegar

Copiá todo el bloque siguiente — incluidos los `---` de apertura y cierre — en Global o Project Instructions.

```
---
LOG-TO-ASANA — registrar trabajo en Asana

Al final de CADA sesión o scheduled task en este workspace, si produjiste al menos un entregable concreto vía MCP (página Notion create/update, evento Calendar create/update, archivo Drive, tarea Asana creada por el usuario, workflow n8n, diseño Canva), creá una tarea de tracking en Asana. Si la sesión fue solo lectura/búsqueda/conversación, no hagas nada.

Cargá SIEMPRE la tarea, incluso si la ejecución quedó trunca — usá el campo `completed` para diferenciar:
- completed=true si la sesión terminó sin errores, sin TODOs pendientes, sin estado BLOCKED.
- completed=false si hubo error no resuelto, timeout, interrupción, o quedó algo a medias.

Pasos:

1. RUTEO
   - Extraé una keyword del entregable (cliente, proyecto, tema).
   - asana_typeahead_search(resource_type="project", query=<keyword>).
   - Match top-1 claro → usá ese gid.
   - Sin match o ambiguo → fallback al proyecto "Inbox Claude" (gid 1214869577300320).

2. CREAR TAREA con asana_create_task:
   - name: "<verbo + objeto>" sin duración. Ej: "Generar reporte semanal de clientes", "Sincronizar pedidos NIKSAN".
   - notes (markdown):
       **Resumen:** <2-3 líneas describiendo el trabajo y, si quedó trunca, qué falta>
       **Artefactos:** lista con bullets (título página + url, evento + url, archivo, etc.)
       **Sesión:** Claude Cowork · <YYYY-MM-DD HH:MM> → <HH:MM> (<duración>)
       **Trigger:** scheduled task: "<nombre>" | manual
   - projects: [<project_gid>]
   - assignee: "me"
   - completed: true | false (según regla arriba)
   - start_at: ISO 8601 UTC del inicio de sesión
   - due_at: ISO 8601 UTC del momento actual
   - custom_fields: {"1213262053852171": <minutos>, "1213322378870926": <minutos>}
     (Estimated time + Actual time del workspace Uchina Studio. Si el proyecto no los tiene, la llamada los omite sin fallar.)

3. CONFIRMAR
   - Si la creación es exitosa: respondé brevemente con el permalink_url devuelto y el estado (Completada / En progreso).
   - Si falla: reportá el error en tu mensaje de cierre. No reintentes (evita duplicados).

Contexto del workspace: Uchina Studio (gid 1213322073728182), team Uchina Studio (1213322073728184). Tag manual a aplicar después si querés: "Claude-Code" (gid 1214864371147865).
---
```

## Notas

- **Tokens**: este bloque ocupa ~650 tokens. En Global Instructions se carga en cada sesión de Cowork. Si te resulta mucho, movelo a Project Instructions de proyectos específicos.
- **Tag Claude-Code**: el MCP de Asana no expone asignación de tags. Si querés que las tareas creadas desde Cowork también lo tengan, podés correr una limpieza periódica desde Code con el PAT (la skill del plugin lo hace al final si el archivo `~/.claude/secrets/asana-pat` existe).
- **Sincronización futura**: si actualizamos el comportamiento de la skill principal, hay que actualizar este bloque manualmente y reemplazarlo en Cowork. Está duplicado a propósito por el límite arquitectónico de Cowork.
