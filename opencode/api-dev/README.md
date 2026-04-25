# api-dev — OpenCode Commands

Senior Rails developer commands for `core-web-api`. Port de los skills de Claude Code al formato de comandos de OpenCode.

## Comandos disponibles

| Comando | Descripción | Modelo |
|---|---|---|
| `/api-dev-plan <notion-url>` | Genera un plan de implementación desde una tarea de Notion | claude-opus-4-7 |
| `/api-dev-execute <plan.md>` | Ejecuta un plan: TDD RED→GREEN + rubocop + undercover | claude-sonnet-4-6 |
| `/api-dev-pr-signoff [notion-url]` | Squash + bin/ci + actualiza Notion a Code Review | claude-sonnet-4-6 |
| `/api-dev-ship <notion-url>` | Orquestador completo: plan → checkpoint → execute → checkpoint → PR → signoff | claude-opus-4-7 |

## Instalación

### 1. Copiar los comandos al proyecto

```bash
mkdir -p /ruta/a/core-web-api/.opencode/commands
cp commands/*.md /ruta/a/core-web-api/.opencode/commands/
```

### 2. Copiar los archivos de contexto al proyecto

Los comandos leen contexto desde `.opencode/context/`. Copiar desde el plugin de Claude Code:

```bash
mkdir -p /ruta/a/core-web-api/.opencode/context
cp ../../api-dev/context/*.md /ruta/a/core-web-api/.opencode/context/
```

Los 7 archivos necesarios son:
- `architecture.md`
- `recorders.md`
- `services.md`
- `controllers.md`
- `testing.md`
- `database.md`
- `rubocop.md`

### 3. Configurar MCP de Notion

Los comandos `plan`, `pr-signoff` y `ship` llaman a `mcp__notion__API-retrieve-a-page` y `mcp__notion__API-get-block-children`. Configurar el MCP de Notion en `opencode.jsonc`:

```json
{
  "mcp": {
    "notion": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@notionhq/notion-mcp-server"],
      "env": {
        "OPENAPI_MCP_HEADERS": "{\"Authorization\": \"Bearer <NOTION_TOKEN>\"}"
      }
    }
  }
}
```

## Uso

```bash
# Flujo completo desde una tarea de Notion
/api-dev-ship https://www.notion.so/workspace/Task-abc123

# Solo planificar (sin implementar)
/api-dev-plan https://www.notion.so/workspace/Task-abc123

# Ejecutar un plan ya generado
/api-dev-execute plan-KODIM-456-add-item-variants.md

# Firmar un PR ya abierto
/api-dev-pr-signoff
```
