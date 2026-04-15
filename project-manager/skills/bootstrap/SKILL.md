---
description: One-time setup for the Kathy PM plugin. Initializes the external project-state file (~/.kathy-pm/project-state.md or $KATHY_PM_STATE_PATH) from the template, then ensures the Kathy Fizzy board has the canonical columns and tag taxonomy. Idempotent — safe to re-run.
model: sonnet
---

# Bootstrap — state file + Fizzy board setup

## Input

No arguments. Operates on the `Kathy` board and initializes the external state file.

---

## Step 1 — Load context

Read:
- `${CLAUDE_PLUGIN_ROOT}/context/fizzy-conventions.md`
- `${CLAUDE_PLUGIN_ROOT}/context/project-state.template.md` — the seed for the external state file.

---

## Step 2 — Resolve external state path

Resolution order:
1. If env var `KATHY_PM_STATE_PATH` is set, use that.
2. Else use `$HOME/.kathy-pm/project-state.md`.

Use `Bash` to check:
```bash
path="${KATHY_PM_STATE_PATH:-$HOME/.kathy-pm/project-state.md}"
echo "resolved_path=$path"
echo "exists=$( [ -f "$path" ] && echo yes || echo no )"
```

Report the resolved path to the user.

---

## Step 3 — Initialize state file (if missing)

If the file does **not** exist:

1. Create the parent directory if needed: `mkdir -p "$(dirname "$path")"`.
2. Copy the template content from `${CLAUDE_PLUGIN_ROOT}/context/project-state.template.md` to the resolved path. Use `Write` (preferred) pointing at the absolute resolved path.
3. Report: `"Creé project-state.md en <path>. Este archivo es tu memoria del proyecto; sobrevive a `claude plugin update`."`.

If the file **does** exist, leave it. Report: `"project-state.md ya existe en <path>. No lo toco."`.

---

## Step 4 — MCP tool resolution probe

Pick the Fizzy MCP prefix:

- Try `mcp__1mcp__Fizzy_1mcp_list_boards` first.
- If not available, fall back to `mcp__claude_ai_1MCP__Fizzy_1mcp_list_boards`.
- If neither is available, stop: `"La MCP de Fizzy no está conectada. Verificá 1mcp y reintentá."`.

Cache which prefix works for the rest of this session.

---

## Step 5 — Locate the Kathy board

Call `list_boards`. Find `name == "Kathy"`.

- If not found → stop and ask: `"No encontré un board llamado 'Kathy'. ¿Lo creo via create_board o preferís crearlo manualmente?"` and **wait**.
- If found → cache the board ID.

---

## Step 6 — Inspect current state (parallel)

- `list_columns(board_id)`
- `list_tags(board_id)`

Compare against canon in `fizzy-conventions.md`:

**Columns (in order):** Backlog · Triage · This Week · In Progress · Review / Testing · Done

**Tags (flat, prefixed):**

- Type: `type:feature`, `type:bug`, `type:chore`, `type:research`, `type:spike`, `type:ops`
- Priority: `P0-critical`, `P1-high`, `P2-medium`, `P3-low`
- Area: `area:whatsapp`, `area:sales-flow`, `area:booking-flow`, `area:menu-catalog`, `area:prompt`, `area:admin-ui`, `area:customer-ui`, `area:billing`, `area:infra`, `area:analytics`
- Repo: `repo:api`, `repo:admin`, `repo:customer`, `repo:cross`, `repo:none`
- Size: `size:XS`, `size:S`, `size:M`, `size:L`
- Flags: `blocked`, `needs-input`

Sprint tags (`sprint:YYYY-Www`) are **not** pre-created — `weekly-plan` makes them on demand.

---

## Step 7 — Show the diff

```
### Bootstrap plan

External state: <path>  (<exists | created from template>)
Board: Kathy (id: <id>)

Columns:
  ✅ already exists: <list>
  ➕ to create:     <list>
  ⚠️ present but not in canon (left untouched): <list>

Tags:
  ✅ already exists: <list>
  ➕ to create:     <list>
  ⚠️ present but not in canon (left untouched): <list>

Nothing gets deleted.
```

If nothing to create in Fizzy → `"Board ya configurado."` and stop.

---

## Step 8 — Confirm

Ask: `"¿Creo los elementos ➕? (sí / solo columnas / solo tags / no)"` and **wait**.

---

## Step 9 — Apply

On confirmation:

- Columns: `create_column(board_id, name, position)` preserving canonical order.
- Tags: use the MCP's tag creation mechanism. If Fizzy only creates tags by tagging a card, create a placeholder card `[bootstrap] tag seed` in Backlog, apply every missing tag via `toggle_card_tag`, then `close_card` with a comment `"tags seeded — card safe to archive"`.

Stop on any error. Do not continue.

---

## Step 10 — Final report

```
### Bootstrap done

External state: <path>
Columns:  ✅ <n> created, <n> already existed
Tags:     ✅ <n> created, <n> already existed

Next steps:
  /project-manager:roadmap       — si aún no definiste el objetivo del mes
  /project-manager:weekly-plan   — si querés arrancar el sprint de esta semana
  /project-manager:groom         — si ya hay cards en Backlog
```

Do **not** touch `project-state.md` content beyond the initial copy from template.

---

## Hard rules

- **Idempotent.** Re-running does not create duplicates nor overwrite the external state file.
- **Never overwrite an existing state file.** Only create if missing.
- **Never delete** columns or tags, even non-canonical ones.
- **Ask before writing** to Fizzy.
