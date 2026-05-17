# log-to-asana

Plugin para Claude Code, Claude Cowork y Claude Chat que registra automáticamente el trabajo realizado como tareas trazables en Asana.

## Qué hace

Al finalizar una sesión donde produjiste algo concreto (archivos creados/editados, commits, specs, workflows n8n, páginas Notion, eventos de Calendar, etc.), la skill `log-to-asana`:

1. **Detecta** los entregables producidos.
2. **Rutea** la tarea al proyecto correspondiente en Asana (mapping local opcional + typeahead search como fallback).
3. **Arma** una tarea con título descriptivo, descripción estructurada, fechas (start + due), tag `claude-code`, y custom fields `Estimated time` / `Actual time` cuando estén disponibles.
4. **Confirma** con el usuario (modo interactivo o con timeout corto) o se ejecuta automáticamente (scheduled tasks de Cowork, hook Stop de Code).

Si no hubo entregables concretos (sesión solo de lectura/búsqueda/conversación), no crea nada.

## Instalación

### Opción A — Marketplace en Claude Code

```bash
# Agregar este repo como marketplace
/plugins marketplace add Akinomotto/claude-log-to-asana

# Instalar el plugin
/plugins install log-to-asana@akinomotto-marketplace
```

### Opción B — Marketplace en Claude Cowork

En Settings → Plugins → Add marketplace, agregá `Akinomotto/claude-log-to-asana`. Después instalá `log-to-asana` desde la lista.

## Requisitos previos

- **MCP de Asana conectado** (`asana@claude-plugins-official` u otro). La skill llama a las tools `asana_create_task`, `asana_typeahead_search`, `asana_get_project`, etc.
- **Permisos en Asana** para crear tareas y leer proyectos del workspace donde quieras registrar.

## Setup automático por entorno

### Claude Code — hook Stop con auto-confirm

Para que la skill se invoque automáticamente al cerrar cada sesión de Claude Code, agregá este hook a `~/.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "INPUT=$(cat); SESSION_ID=$(echo \"$INPUT\" | jq -r .session_id); MARKER=\"/tmp/.log-to-asana-fired-${SESSION_ID}\"; if [ -f \"$MARKER\" ]; then exit 0; fi; touch \"$MARKER\"; echo '{\"decision\":\"block\",\"reason\":\"Antes de cerrar esta sesión, invocá el skill log-to-asana en modo --auto-with-confirm-timeout=5 para registrar el trabajo realizado en Asana. Si la skill no detecta entregables, saldrá silenciosamente sin crear nada.\"}'",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

El hook tiene loop-guard (marker en `/tmp/.log-to-asana-fired-<session_id>`) para no dispararse infinitamente.

### Claude Cowork — Global Instructions

Pegá este bloque en `Settings → Cowork → Edit Global Instructions` (una sola vez para todo el workspace):

> **Al finalizar cualquier scheduled task de este workspace, invocá el skill `log-to-asana` en modo `--auto` pasándole como contexto el nombre de la scheduled task. La skill detectará si hubo entregables productivos (archivos creados, páginas Notion, eventos de Calendar, workflows n8n, etc.) y registrará el trabajo en Asana automáticamente en el proyecto correspondiente. Si no hubo entregables (la sesión fue solo lectura/búsqueda/conversación), no hará nada.**

Para activarlo solo en algunos Projects internos de Cowork (no en todo el workspace), pegá el mismo bloque en las Project Instructions de los proyectos elegidos.

### Claude Chat (claude.ai)

Invocación manual con `/log-to-asana` o frases naturales: "registrá esto en Asana", "cargá esto a Asana", "anotá en Asana lo que hicimos".

## Mapping local opcional (solo Code)

Si querés que la skill rutee a proyectos específicos sin depender solo de typeahead, creá `~/.claude/asana-project-mapping.json`:

```json
{
  "default_project_gid": "<gid del proyecto fallback Inbox Claude>",
  "default_project_name": "Inbox Claude",
  "workspace_gid": "<gid del workspace>",
  "rules": [
    {
      "match": { "keywords": ["mi-cliente", "alias-cliente"] },
      "project_gid": "<gid del proyecto>",
      "project_name": "Mi Cliente"
    },
    {
      "match": { "path_glob": "**/audit-*" },
      "project_gid": "<gid>",
      "project_name": "Audits"
    }
  ]
}
```

La skill puede sugerir agregar reglas nuevas a este mapping cuando ruteás manualmente a un proyecto que no estaba.

## Proyecto fallback "Inbox Claude"

Si la skill no puede resolver un proyecto, crea (o usa si ya existe) un proyecto llamado "Inbox Claude" en el workspace. Revisalo periódicamente y movelo las tareas a su proyecto definitivo.

## Custom fields opcionales

Si tu workspace tiene los custom fields `Estimated time` / `Actual time` (en/es: `Tiempo estimado` / `Tiempo real`) asociados al proyecto destino, la skill los pobla automáticamente en minutos. `Estimated time` es built-in de Asana en todos los proyectos; `Actual time` se asocia manualmente por proyecto (la MCP no permite hacerlo; pero ver §PAT abajo para bootstrap masivo).

## PAT opcional (acceso avanzado a Asana)

El MCP de Asana cubre la mayoría de operaciones pero no todas: por ejemplo, **agregar tags a tareas existentes**, **asociar custom fields a proyectos**, o **crear tags nuevos** requieren llamadas REST directas.

Para habilitarlas, generá un Personal Access Token en https://app.asana.com/0/my-apps y guardálo así:

```bash
mkdir -p ~/.claude/secrets && chmod 700 ~/.claude/secrets
echo -n "tu-pat-acá" > ~/.claude/secrets/asana-pat
chmod 600 ~/.claude/secrets/asana-pat
```

La skill detecta el archivo automáticamente y lo usa para esas operaciones. Sin el archivo, la skill omite silenciosamente esas operaciones (no falla). Si revocás el PAT, las operaciones que lo requieran van a fallar — generá uno nuevo y reemplazá el contenido del archivo.

**Casos resueltos con PAT:**
- Aplicación automática del tag `Claude-Code` a cada tarea creada.
- Bootstrap del custom field `Actual time` en todos los proyectos de un workspace (one-liner script en el repo).

## Licencia

MIT — ver `LICENSE`.
