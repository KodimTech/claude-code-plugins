---
description: Plan the current week's sprint for Kathy. Reads external project-state.md (Rolling capacity + Current objective), surveys Triage, proposes cards up to the calibrated capacity with narrative per-card verdicts (no numeric score), tagged with sprint:YYYY-Www. Use every Monday.
model: opus
---

# Weekly plan — Kathy sprint

## Input

Optional: `$ARGUMENTS` may contain a specific ISO week (e.g., `2026-W16`) or a one-off capacity override (e.g., `capacity=4`). If absent, compute the ISO week from today and use the Rolling capacity.

---

## Step 1 — Load context (parallel)

Read:
- `${CLAUDE_PLUGIN_ROOT}/context/kathy.md`
- `${CLAUDE_PLUGIN_ROOT}/context/fizzy-conventions.md`
- `${CLAUDE_PLUGIN_ROOT}/context/pm-playbook.md`
- `${CLAUDE_PLUGIN_ROOT}/context/domain-heuristics.md`

---

## Step 2 — Resolve and read external state

Resolve state path (`$KATHY_PM_STATE_PATH` or `~/.kathy-pm/project-state.md`). If missing → stop: `"No encuentro project-state.md. Corré /project-manager:bootstrap primero."`.

Read from the external file:
- `Current objective`
- `Active sprint` (check if one is already open)
- `Rolling capacity § Current capacity` — this is the planning budget
- `Known blockers`
- `Open questions for the founder`
- `Customer feedback inbox` — anything waiting to become a card

If there is already an active sprint that is not `closed`, stop: `"Ya hay un sprint abierto (<tag>, estado <status>). Cerralo primero con /project-manager:weekly-review o pediblo explícitamente."`.

---

## Step 3 — MCP + board

- Resolve Fizzy MCP prefix (`mcp__1mcp__Fizzy_1mcp_*` with `mcp__claude_ai_1MCP__*` fallback).
- `list_boards` → find "Kathy", cache `board_id` and column IDs.

Compute sprint tag: `sprint:YYYY-Www` (ISO week of today or `$ARGUMENTS`).
Compute window: Monday → Sunday of that week.

---

## Step 4 — Resolve capacity for this sprint

Priority:
1. One-off override in `$ARGUMENTS` (e.g., `capacity=4`).
2. `Current capacity` from `project-state.md`.
3. Fallback to 10h if state doesn't yet have a value.

Report the source: `"Capacidad para este sprint: Xh (fuente: rolling / override / seed)"`.

Reserve **1h buffer**. Target planned hours = `capacity - 1`.

---

## Step 5 — Survey the board (parallel)

- `list_column_cards(board_id, "Triage")` — candidate pool.
- `list_column_cards(board_id, "This Week")` — spillover from prior sprint.
- `list_column_cards(board_id, "In Progress")` — active, eats capacity.
- `list_column_cards(board_id, "Review / Testing")` — likely to close this week.

For each, pull `get_card` if metadata is missing.

---

## Step 6 — Filter Triage for Definition of Ready

From `pm-playbook.md § Definition of Ready`. A card is READY when it has:
- `type:*`, `P*`, `area:*`, `repo:*`, `size:*`
- Not `size:L`
- Not `needs-input`
- Not `blocked` (or blocked with a clear unblock path in comments)
- Full description sections (or ops variant)

NOT-READY cards are listed separately so the user can decide whether to `/project-manager:groom` first.

---

## Step 7 — Rank via narrative verdicts (no numeric score)

For each READY card, write a 3-line verdict:

```
Card #<id> — "<title>"
  Value: <why it matters — tie to objective / 30-customer goal / domain heuristic>
  Cost:  <size + any realistic risk multiplier>
  Risk:  <low / medium / high — and why>
  Verdict: incluir / incluir si cabe / dejar en Triage / devolver a Backlog
```

**Decisive filters** (applied during verdict writing):
- Card violates a `domain-heuristics.md` rule → note the reference and recommend defer unless the founder has a good reason.
- Card carried over from last sprint more than once → flag explicitly: *"Se arrastra desde sprint-N. ¿Sigue siendo prioridad o volvemos a Triage?"*. Do not auto-promote.
- Card has `blocked` tag with no clear unblock path → exclude.

---

## Step 8 — Fit the sprint

Count midpoint-hours (XS=0.75, S=2, M=4.5) from `fizzy-conventions.md § Size`.

Start with:
- Active work: `In Progress` + `Review / Testing` sum first (counts against capacity).
- Then fill with "incluir" verdicts in order of strength.

Stop adding when next card pushes past `capacity - 1h buffer`.

**Must-haves:**
- At least one card tied to `Current objective` OR the 30-customer goal (per `kathy.md` + `domain-heuristics.md`). If the READY pool has none that qualify, **stop**: `"Ninguna card READY está atada al objetivo. ¿Hacemos /project-manager:groom primero o ampliamos Triage?"`.
- Consider including at least one **ops card** (`type:ops`) if `project-state.md § Known blockers` suggests one (e.g., Meta follow-up pending 2+ weeks). An all-code sprint while ops rot is usually a planning bug.

---

## Step 9 — Propose the sprint (narrative)

```
### Plan semanal — sprint:<YYYY-Www>

Ventana: <YYYY-MM-DD> → <YYYY-MM-DD>
Capacidad calibrada: <Xh>  (fuente: <rolling / override / seed>)
Objetivo del mes: <de project-state.md>

Ya en marcha (descuenta capacidad):
  - [<size> · <area> · <repo>] #<id> <title>   — estado: <In Progress / Review>
  - ...
  Subtotal activo: <Yh>

Propuesta para This Week (nuevas):

  1. Card #<id> — "<title>"
     Value: <…>
     Cost:  <size · Xh>
     Risk:  <…>
     Verdict: incluir. <una oración de por qué encaja ahora.>

  2. Card #<id> — "<title>"
     Value: <…>
     Cost:  <size · Xh>
     Risk:  <…>
     Verdict: incluir si cabe. <…>

  ...

  Total planificado: <Zh> / <capacity>h (margen <b>h)

Atadas al objetivo: <n>  (mínimo 1 ✓)
Ops incluidas:      <n>

Carried-over (preguntar antes de planificar):
  - #<id> "<title>" — ya se arrastra desde sprint-<N>. ¿Seguimos o la volvemos a Triage?

NOT READY (dejadas en Triage):
  - #<id> "<title>" — falta: <qué>

Riesgos y dependencias:
  - <p.ej. "Card #3 depende de migración de #1">
  - <p.ej. "area:whatsapp sigue gated por Meta — card #4 puede convertirse en blocked">

Preguntas abiertas para vos:
  - <list or "ninguna">
```

---

## Step 10 — Confirm

Ask: `"¿Aplico este plan? (sí / edítalo / cambia X por Y / no)"` and **wait**.

If the user edits, recompute the table and re-show. Iterate until **sí**.

---

## Step 11 — Apply

Only after explicit "sí":

For each card in the final plan:
1. `toggle_card_tag(card_id, "sprint:YYYY-Www")` (creates the tag on first toggle if it doesn't exist; verify via `list_tags` after the first toggle).
2. Move to `This Week` column via `update_card`.
3. Append comment: `"Incluida en sprint <YYYY-Www> (plan del <YYYY-MM-DD>)."`.

Run sequentially per card so the log is readable.

---

## Step 12 — Update external `project-state.md` (Active sprint section only)

Edit the external state file — only the `Active sprint` block:

- `Tag`: `sprint:YYYY-Www`
- `Window`: `YYYY-MM-DD → YYYY-MM-DD`
- `Planned`: midpoint sum
- `Status`: `started`

Do **not** touch `Rolling capacity`, `Recent decisions`, `Retro — last sprint`, or `Roadmap snapshot` here. Those belong to `weekly-review` and `roadmap`.

---

## Step 13 — Report

```
### Sprint abierto: sprint:<YYYY-Www>

Cards en This Week:    <n>
Horas planificadas:    <X>h / <capacity>h  (margen <b>h)
Cards atadas al objetivo:  <n>
Cards ops:             <n>

Sugerencia para hoy: empezar por #<card con mayor Value/lower Risk>.

Siguiente paso técnico: para cada card dev, abrir el repo correspondiente y correr:
  /api-dev:plan <card-id-o-descripción>       (repo:api)
  /customer-dev:plan <card-id-o-descripción>  (repo:admin | repo:customer)

Mid-week:  /project-manager:status
Viernes:   /project-manager:weekly-review
```

---

## Hard rules

- **Never auto-include without a verdict.** Every proposed card has a 3-line story.
- **Never exceed calibrated capacity silently.** If the founder insists, log the risk in the sprint comment.
- **Never include an L card.** Tell the user to split.
- **Always at least one card tied to the objective**, or stop.
- **Every moved card gets a comment.** No silent transitions.
- **Read external `project-state.md`** — never the plugin's template.
- **Every Fizzy call respects MCP prefix resolution** (`1mcp` first, `claude_ai_1MCP` fallback).
