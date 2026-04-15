---
description: Short mid-week status of the active Kathy sprint. Read-only. Shows what's in progress, in review, what closed, what's blocked, and how the sprint tracks against the calibrated capacity from project-state.md. Does not modify cards or state.
model: sonnet
---

# Status — active sprint snapshot

## Input

No arguments.

---

## Step 1 — Load minimal context

Read:
- `${CLAUDE_PLUGIN_ROOT}/context/fizzy-conventions.md` — to know the size midpoints.

---

## Step 2 — Resolve external state

Resolve `project-state.md`. If missing → stop and hint at `/project-manager:bootstrap`.

Read:
- `Active sprint` (tag, window, planned, status)
- `Rolling capacity § Current capacity`

If no active sprint or status is `closed`:

```
No hay sprint activo. Corré /project-manager:weekly-plan para abrir el de esta semana.
```

Stop.

---

## Step 3 — MCP + board

Resolve Fizzy MCP prefix. `list_boards` → "Kathy", cache `board_id`.

---

## Step 4 — Pull sprint cards

- `list_cards(board_id, tag=<sprint>)` (or column-by-column fallback if MCP doesn't filter by tag).
- For each, `get_card` for tags, column, latest comments.

Group by column: `This Week`, `In Progress`, `Review / Testing`, `Done`.
Also pull any card with `blocked` tag regardless of sprint — the founder should see the stuck stuff.

---

## Step 5 — Compute sprint metrics (using midpoints)

From `fizzy-conventions.md`:
- XS → 0.75h
- S → 2h
- M → 4.5h
- (L excluded — shouldn't be in a sprint; if detected, flag it)

Compute:
- `planned_h` = sum of midpoints across all cards tagged with the active sprint.
- `closed_h` = sum of midpoints across cards currently in `Done`.
- `completion_pct` = `closed_h / planned_h`.
- `days_elapsed` vs `days_total` in window.
- `wip_count` = cards in `In Progress`. Target ≤1, max 2.

---

## Step 6 — Output (one screen, read-only)

```
### Status — sprint:<YYYY-Www>

Ventana: <YYYY-MM-DD> → <YYYY-MM-DD>   (día <N>/7)
Capacidad calibrada: <Xh> · Planificado: <Y>h · Cerrado: <Z>h (<W>%)
WIP In Progress: <n>   (recomendado ≤1, máximo 2)

In Progress:
  - [<size·midpoint>h · <area> · <repo>] #<id> <title>   (iniciada hace <n>d)

Review / Testing:
  - [<size·midpoint>h] #<id> <title>   (esperando <QA / PR / merge>)

This Week (no empezadas):
  - [<size·midpoint>h · P<n>] #<id> <title>

Done este sprint:
  - #<id> <title>
  - #<id> <title>

Bloqueadas (tag `blocked`, dentro o fuera del sprint):
  - #<id> <title> — razón: <quote del comentario>

Señales a mirar:
  - <p.ej. "Card #X lleva 4 días en In Progress con tamaño S (2h). Señal: algo técnico no previsto — vale pedir /api-dev:plan o abrir ops card de investigación.">
  - <p.ej. "Sprint al 20% con 2 días restantes. Opciones: recortar AC en #Y, mover #Z a próxima semana, o aceptar spillover.">
  - <p.ej. "WIP=3: rompé la regla. Cerrá una card antes de arrancar otra — el switching cost te está costando el sprint.">

Sugerencia concreta para hoy:
  <una sola acción>
```

---

## Hard rules

- **Read-only.** No card mutations, no tag toggles, no comments, no column moves.
- **Do not update `project-state.md`.**
- **Single sprint focus** — only cards with the active sprint tag.
- **Surface 1-3 signals**, not a wall. The founder should close this in 30 seconds.
- **Honest numbers.** Use midpoints consistently with `fizzy-conventions.md § Size`.
