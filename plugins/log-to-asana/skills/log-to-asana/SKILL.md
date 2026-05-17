---
name: log-to-asana
description: Use when finishing a significant piece of work in Claude (Code, Cowork or Chat) that produced concrete deliverables (files created/edited, commits, specs, n8n workflows, Notion pages, Calendar events) and the user wants it tracked as a task in Asana. Triggers on phrases like "log to asana", "registrá esto en Asana", "cargá esto a Asana", "anotá en Asana lo que hicimos", or when invoked as /log-to-asana. Also invoked automatically by the Stop hook of Claude Code at session end (with 5s confirm timeout) and by Cowork Global Instructions at the end of every scheduled task (in --auto mode).
---

# log-to-asana

Registra el trabajo realizado en Claude como tareas trazables en Asana. Funciona en Claude Code (hook automático + invocación manual), Claude Cowork (vía Global Instructions del workspace) y Claude Chat (invocación manual con slash o frase natural).

**Requisito previo:** el MCP de Asana debe estar conectado (plugin `asana@claude-plugins-official` o equivalente). La skill llama tools como `asana_create_task`, `asana_typeahead_search`, `asana_get_project`.

## Detección de entregables

Al ser invocada, analizá la sesión actual y determiná si hay entregables que justifiquen crear una tarea en Asana.

**Cuenta como entregable (al menos una de estas):**

- Uno o más archivos creados o editados con Edit/Write, excluyendo:
  - `MEMORY.md` y archivos bajo `~/.claude/memory/`
  - Archivos de sistema (`.DS_Store`)
  - Archivos bajo `/tmp/`
- Uno o más commits nuevos hechos durante la sesión.
- Specs o documentos de diseño nuevos.
- Un workflow de n8n creado/modificado vía MCP (`n8n_create_workflow`, `n8n_update_full_workflow`, `n8n_update_partial_workflow`).
- Una llamada exitosa a un MCP "productivo":
  - Notion: `notion-create-pages`, `notion-update-page`, `notion-create-database`
  - Calendar: `create_event`, `update_event`
  - Drive: `create_file`
  - Canva: `create-design-*`, `generate-design`
  - Otras tareas creadas en Asana por el usuario durante la sesión.

**No cuenta como entregable:**

- Solo lectura/exploración: Read, Grep, Glob, búsquedas web, lecturas de MCP (`notion-fetch`, `notion-search`, `get_*`).
- Sesiones de brainstorming sin spec escrito al cierre.
- Sesiones donde el único output fue conversación.

**Si no hay entregable detectado:**
- Modo interactivo: informá al usuario "No detecté entregables cargables en esta sesión" y terminá.
- Modo `--auto` o `--auto-with-confirm-timeout`: salí silenciosamente sin crear nada ni loguear error.

**Agrupación multi-proyecto:**
Si los entregables se asocian a más de un proyecto Asana (ver §Ruteo), creá **una tarea por proyecto**, cada una con sus artefactos correspondientes. No crees tareas cruzadas.

## Ruteo a proyectos

El destino de cada tarea se resuelve por dos vías: mapping local (si existe) y typeahead search (siempre disponible).

### Paso 1 — Intentar leer mapping local (opcional)

Si estás en Claude Code, intentá leer `~/.claude/asana-project-mapping.json`. Si existe y es JSON válido, usalo como conjunto de reglas determinísticas de ruteo. Formato esperado:

```json
{
  "default_project_gid": "<gid del proyecto fallback Inbox Claude>",
  "default_project_name": "Inbox Claude",
  "workspace_gid": "<gid del workspace>",
  "rules": [
    { "match": { "keywords": ["foo", "foobar"] }, "project_gid": "<gid>", "project_name": "Foo" },
    { "match": { "path_glob": "**/audit-*" }, "project_gid": "<gid>", "project_name": "Audits" }
  ]
}
```

Si no existe (caso normal en Cowork/Chat) o está corrupto, saltá al Paso 2.

### Paso 2 — Resolver vía typeahead search

Para cada grupo de entregables, detectá keywords candidatas (nombres de archivos tocados, palabras clave del transcript, nombre del repo git si aplica). Para cada keyword:

1. Llamá a `asana_typeahead_search` con `resource_type=project` y la keyword como query.
2. Si devuelve un resultado claro (top-1 con match obvio del nombre), usalo como candidato.
3. Si devuelve varios o ninguno, marcá el grupo como "ambiguo".

### Paso 3 — Resolver el proyecto final

- **Match determinístico** (mapping local o typeahead unívoco): usá ese proyecto sin preguntar.
- **Match ambiguo o sin match:**
  - Modo interactivo y modo confirm-timeout: proponé el top-1 con razonamiento explícito ("creo que va a NombreProyecto porque tocaste `archivo...`") y ofrecé alternativas + opción de buscar otro proyecto del workspace via `asana_typeahead_search`.
  - Modo `--auto`: caé al proyecto fallback "Inbox Claude" (si existe en el workspace). Si no existe, creálo via `asana_create_project` con team default y reusá el gid.

### Paso 4 — Aprendizaje incremental (solo modo interactivo en Code)

Si estás en Code, el mapping local existe, y el usuario eligió un proyecto que no estaba en las reglas, preguntale si querés agregar la regla al mapping. Si dice que sí:
- Identificá el patrón que tendría que haber matcheado (path glob o keyword).
- Agregá la regla nueva al final del array `rules` en el mapping.
- Validá que el JSON siga siendo válido.

## Armado de la tarea

Para cada proyecto resuelto:

### Título

Patrón: `<verbo + objeto> · <duración>`. El verbo describe la acción principal, el objeto identifica el output. La duración va al final separada por `·`.

Ejemplos genéricos:
- `Auditar website de cliente · 45min`
- `Crear spec de integración · 1h20`
- `Refactorizar workflow n8n · 30min`

Formato de duración:
- `< 1h` → `XXmin` (ej.: `45min`)
- `≥ 1h` → `XhYY` (ej.: `1h20`, `2h05`)

### Descripción (campo `notes`)

Markdown con esta plantilla:

```
**Resumen:** <2-3 líneas describiendo qué se hizo>

**Artefactos:**
- <path/al/archivo1.md> (creado)
- <path/al/archivo2.html> (editado)
- commit <abc1234>: <mensaje del commit>
- Notion: <título de la página> — <url>

**Sesión:** Claude <Code|Cowork|Chat> · <YYYY-MM-DD HH:MM> → <HH:MM> (<duración>)
**Trigger:** <manual | hook | sugerencia | scheduled task: "<nombre>">
```

Incluí solo las líneas de artefactos que apliquen. Para commits, incluí el hash corto y el mensaje. Para entradas de MCPs, incluí título y URL si está disponible.

### Tiempo

- `start`: timestamp del primer mensaje del usuario en la sesión actual (hora local).
- `end`: timestamp del momento en que se invoca la skill.
- `duración`: diferencia formateada según el patrón de arriba.

### Subtareas

Si la sesión produjo varios entregables dentro del **mismo** proyecto que son separables (ej.: spec + workflow + commit), creá:
- Una tarea madre con resumen general y duración total.
- Subtareas por cada entregable significativo, usando `asana_create_task` con `parent` apuntando a la tarea madre.

Heurística para subdividir: si los artefactos pertenecen a tipos distintos (spec, workflow, código) o están separados por commits diferentes, subdividí. Si todos son ediciones al mismo archivo o área lógica, no.

### Campos de la tarea

Al llamar `asana_create_task`:
- `name`: título según patrón.
- `notes`: descripción según plantilla.
- `projects`: `[project_gid]` resuelto en §Ruteo.
- `assignee`: `me` (o el gid del usuario actual).
- `start_at`: timestamp ISO 8601 del inicio de la sesión (UTC).
- `due_at`: timestamp ISO 8601 del momento de carga de la tarea (UTC).
- `completed`:
  - **Modo interactivo:** preguntá al usuario en el prompt de confirmación si la tarea está completada (`¿completada? [s/N]` adicional al confirm normal). Si responde `s` → `true`; si `N` o sin respuesta → `false`.
  - **Modo `--auto` y `--auto-with-confirm-timeout`:** asumí `completed: true` (la routine/sesión cerró exitosamente). Excepción: si se detecta que quedaron TODOs pendientes, tests fallando o estado BLOCKED → `false`.
- `tags`: incluí el tag `claude-code`. Si no existe en el workspace, creálo primero y reusá el gid.
- `custom_fields`: ver §Custom fields opcionales.

### Custom fields opcionales

Antes de crear la tarea, leé los `custom_field_settings` del proyecto destino (`asana_get_project` con `opt_fields=custom_field_settings.custom_field.name,custom_field_settings.custom_field.gid`). Si encontrás campos con estos nombres (matching case-insensitive, español o inglés), llenálos:

| Nombre buscado | Valor a poblar |
|---|---|
| `Estimated time` / `Tiempo estimado` | Duración estimada en **minutos** (entero). En modo `--auto`, usá la misma que `Actual time`. En modo interactivo, preguntá al usuario o asumí igual a la real. |
| `Actual time` / `Tiempo real` | Duración real de la sesión en **minutos** (entero). |

Si el campo no está asociado al proyecto destino, omitilo silenciosamente (no fallar). Pasalos en `custom_fields` como JSON: `{"<gid>": <minutos>, ...}`.

**Nota sobre `Estimated time` built-in:** Asana tiene un campo nativo `Estimated time` disponible en todos los proyectos por defecto. Si no aparece en `custom_field_settings`, igual podés intentar poblarlo — Asana lo acepta. Si falla, omitirlo y continuar.

**Nota sobre `Actual time` custom:** este campo lo tiene que asociar el usuario manualmente a cada proyecto donde lo quiera usar (la API de Asana no permite asociar custom fields a proyectos automáticamente vía MCP). El skill detecta su disponibilidad y se adapta.

### Confirmación previa (modo interactivo y confirm-timeout)

Antes de llamar `asana_create_task`, mostrá al usuario un resumen de la propuesta:

```
Detecté N entregable(s) agrupable(s) en M proyecto(s):

→ Proyecto: <nombre> (<motivo del match>)
→ Título: "<título propuesto>"
→ Artefactos: <lista corta>
→ Duración: <duración> (start: <YYYY-MM-DD HH:MM> → end: <HH:MM>)
→ Custom fields detectados: <Estimated time / Actual time / ninguno>

¿Cargar? [Enter]=sí (<N>s) / n=no / e=editar
¿Completada? [s/N]   ← preguntar SOLO en modo interactivo
```

Modo interactivo: espera respuesta sin timeout. Preguntá explícitamente por completed.
Modo confirm-timeout: si no hay respuesta en N segundos o llega `Enter`, crea la tarea (auto-confirm). `n` cancela. `e` modo edición. Asume `completed: true` salvo TODOs pendientes / errores.

Si hay más de un proyecto, mostrá una propuesta por proyecto y confirmá una por una.

### Modo `--auto`

Saltear la confirmación. Crear directamente. Si falla y estás en Claude Code, append al log `~/.claude/logs/asana-sync.log` con timestamp, mensaje de error y datos de la tarea que se intentó crear. En Cowork/Chat (sin filesystem local), reportar el error como mensaje al final de la sesión. Continuar con las demás tareas si hay varias.

## Detección del entorno y modo

Al invocarte, determiná el modo de ejecución:

- Si te invocan con `--auto` (o el contexto indica que sos parte del cierre de una scheduled task de Cowork): **modo `--auto`** — sin confirmación, salida silenciosa, errores al log.
- Si te invocan con `--auto-with-confirm-timeout=N` (uso típico del hook Stop de Claude Code): **modo híbrido** — mostrá la propuesta con prompt `[Enter]=sí (Ns) / n=no / e=editar` y timeout de N segundos a auto-confirm. Si no hay entregable, salí silenciosamente igual que en `--auto`.
- En cualquier otro caso (slash command, frase natural, sugerencia tuya): **modo interactivo** — espera respuesta del usuario sin timeout.

Identificá el entorno (`Code`, `Cowork`, `Chat`) leyendo señales del contexto:
- `Code`: hay tools de Bash, Edit, Write disponibles y se tocaron archivos locales.
- `Cowork`: viene como scheduled task con nombre identificable; usar ese nombre en el campo `Trigger` de la tarea.
- `Chat`: no hay sistema de archivos local; los entregables vienen exclusivamente de MCPs.

El entorno va en el campo `Sesión` de la descripción.

## Manejo de errores

| Situación | Modo interactivo | Modo `--auto` / confirm-timeout |
|---|---|---|
| MCP de Asana no disponible | Avisá al usuario y abortá. | Log y salida con código 1. |
| `project_gid` del mapping inválido | Avisá, caé a Inbox Claude. | Caé a Inbox Claude. |
| Inbox Claude no existe | Ofrecé crearlo. | Creálo automáticamente. |
| Sin entregable | Avisá "no hay nada cargable". | Salí silenciosamente con código 0. |
| Mapping corrupto/ausente | Saltá a typeahead. | Saltá a typeahead. |
| `asana_create_task` falla | Avisá, NO reintentes (puede crear duplicado). | Log, continuá con otras tareas si hay. |

Formato del log (`~/.claude/logs/asana-sync.log`, solo Claude Code):

```
[YYYY-MM-DD HH:MM:SS] ERROR: <mensaje>
  Tarea intentada: <título>
  Proyecto: <nombre> (<gid>)
  Detalle: <stack o mensaje del MCP>
---
```

## Documentación adicional

Para activar el uso automático en scheduled tasks de Claude Cowork (una sola vez para todo el workspace), ver `cowork-global-instruction.md` en este mismo directorio.

Para el hook automático de Claude Code que invoca esta skill al cerrar cada sesión, ver el README del plugin (sección "Setup hook en Claude Code").
