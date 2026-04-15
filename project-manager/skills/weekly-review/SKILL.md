---
description: Close the active Kathy sprint. Summarizes what shipped, what slipped, produces a 3-column retro, updates Rolling capacity from planned-vs-closed, updates project-state.md (Active sprint, Recent decisions, Known blockers, Retro, Rolling capacity). Proposes status changes for spillover cards. Run Fridays.
model: opus
---

# Weekly review — close the sprint

## Input

Optional: `$ARGUMENTS` may contain a specific sprint tag. If absent, use `Active sprint` from external `project-state.md`.

---

## Step 1 — Load context (parallel)

Read:
- `${CLAUDE_PLUGIN_ROOT}/context/kathy.md`
- `${CLAUDE_PLUGIN_ROOT}/context/fizzy-conventions.md`
- `${CLAUDE_PLUGIN_ROOT}/context/pm-playbook.md`
- `${CLAUDE_PLUGIN_ROOT}/context/domain-heuristics.md`

---

## Step 2 — Read external `project-state.md`

Resolve and read the state file. Extract:
- `Active sprint` (tag + window + planned)
- `Rolling capacity` (current value + sample)
- `Known blockers`
- `Current objective`

If no active sprint and no `$ARGUMENTS` → stop: `"No hay sprint activo para cerrar. ¿Me pasás el tag, p.ej. 'sprint:2026-W15'?"`.

---

## Step 3 — MCP + board

Resolve Fizzy MCP prefix. `list_boards` → "Kathy", cache ids.

---

## Step 4 — Pull sprint cards

- `list_cards(board_id, tag=<sprint>)`. For each: `get_card`.
- Group by column: Done / Review-Testing / In-Progress / This-Week / Blocked.

---

## Step 5 — Compute numbers (midpoints from fizzy-conventions)

- `planned_h` = sum midpoints of all sprint-tagged cards.
- `closed_h` = sum midpoints of cards in `Done`.
- `spillover_count`: non-Done cards tagged with this sprint.
- Per-area distribution (hours in Done grouped by `area:*`).
- Per-repo distribution.

---

## Step 6 — Infer reasons for spillover

Per non-Done card, inspect the last 2-3 comments + current column state:

| Signal | Likely reason |
|---|---|
| No comments since sprint start, still in This Week | "nunca arrancó — ¿cambio de prioridad o se olvidó?" |
| In Progress with recent comments mentioning blockers | "trabada por <blocker>" |
| In Review for >2 days | "esperando QA / PR / merge" |
| Tag `blocked` | inspect comment for blocker text |
| Carried over multiple sprints | "arrastra desde sprint-N — señal de que quizá no es lo que parecía" |

---

## Step 7 — Draft the review

```
### Weekly review — sprint:<YYYY-Www>

Ventana:      <YYYY-MM-DD> → <YYYY-MM-DD>
Capacidad:    <Xh>  (calibrada)
Planificado:  <Y>h · Cerrado: <Z>h (<W>%)

Cerradas (<n>):
  - #<id> [<area>·<repo>·<size>] <title>   — outcome: <1 line>
  - ...

Spillover (<n>):
  - #<id> [<area>·<repo>·<size>] <title>
      estado: <In Progress / Review / This Week>
      motivo probable: <inferred>
      sugerencia: <keep in This Week | volver a Triage | partir | cerrar>

Bloqueadas cerradas con bloqueo:
  - #<id> <title> — bloqueo: <quote>

Distribución de tiempo cerrado:
  Área: area:whatsapp <a>h · area:admin-ui <b>h · area:prompt <c>h · ops <d>h
  Repo: api <a>h · admin <b>h · customer <c>h · cross <d>h · none (ops) <e>h

Retro (propuesta — editala antes de guardar):

  🟢 Went well:
    - <concrete thing, ideally citing a domain heuristic that helped>
    - <...>

  🔴 Didn't:
    - <concrete thing>
    - <...>

  🔁 Change next week:
    - <1-2 concrete experiments>

Decisiones registrables (para Recent decisions):
  - <YYYY-MM-DD — decision — why (if load-bearing)>

Bloqueos (update Known blockers):
  - Resueltos: <list>
  - Nuevos:    <list>

Calibración de capacidad:
  Sample actual:  <list of last sprints>
  Nueva entrada:  sprint:<YYYY-Www> — planned <Y>h / closed <Z>h
  Recomputo:      <nueva avg>h  →  Current capacity = round(avg + 1h), clamp [4, 10]
  Cambio:         <Xh → X'h>   (o "sin cambio")
```

---

## Step 8 — Confirm spillover decisions

Ask: `"Editá el retro si querés. Decime cómo manejo el spillover:
  (a) aplico mis sugerencias por defecto
  (b) revisemos uno a uno
  (c) no toco cards, solo actualizo project-state.md"` and **wait**.

Iterate per choice.

---

## Step 9 — Apply spillover

Per approved card:

- **keep en This Week (arrastre)**: leave it there, append comment `"Arrastra a sprint <YYYY-W(w+1)>"`. Do NOT tag the next sprint yet — that's `weekly-plan`.
- **volver a Triage**: `update_card(id, triage_column_id)` + **remove** the sprint tag (`toggle_card_tag` off) + reason comment.
- **partir**: inline split flow — create new cards with minimal description + close original with links.
- **cerrar** (abandonar): `close_card(id)` + reason comment.

---

## Step 10 — Update external `project-state.md` (show diff first)

Proposed edits:

1. `Last updated` → today + `weekly-review`.
2. `Active sprint § Status` → `closed`. Populate `Planned`/`Closed` hours.
3. `Recent decisions` → prepend any new load-bearing decisions from the retro (format: `YYYY-MM-DD — decision — why`).
4. `Known blockers` → add new, remove resolved.
5. `Retro — last sprint` → replace with the 3-column retro from Step 7.
6. `Rolling capacity`:
   - Append `sprint:<YYYY-Www> — planned <Y>h / closed <Z>h` to the sample (newest first).
   - Drop entries older than the last 3.
   - If sample has ≥3 entries: recompute `Current capacity = round(avg(closed) + 1h)`, clamp `[4, 10]`. Update the line.
   - Else: leave `Current capacity` untouched but note the pending calibration.

Show the diff. Ask: `"¿Aplico estos cambios a project-state.md? (sí / editá X / no)"` and **wait**.

On "sí" → apply via `Edit` calls to the external file path.

---

## Step 11 — Report

```
### Review cerrado — sprint:<YYYY-Www>

Cards cerradas:      <n> (<Z>h)
Spillover:           <n>  → <desglose de destinos>
Bloqueos actualizados en project-state.md: <n>
Retro guardado.
Rolling capacity:    <X>h → <X'>h  (o "sin cambio, muestra insuficiente")

Próximos pasos:
  - /project-manager:weekly-plan  — para abrir el próximo sprint con la capacidad nueva
  - /project-manager:roadmap      — si el retro disparó un cambio de dirección
  - Revisar cards que volvieron a Triage: <ids>
```

---

## Hard rules

- **Never close a Done card.** They stay in Done (Fizzy archives).
- **Retro is proposed, never forced.** Founder edits before save.
- **Every spillover card gets a comment** with the reason.
- **Remove the sprint tag** from cards returning to Triage.
- **Recalibrate capacity honestly.** If closed < planned for 3 sprints, the plan lied — shrink.
- **Clamp `Current capacity` to [4h, 10h].** Never auto-raise beyond 10h.
- **Update external `project-state.md`**, never the plugin's template.
- **Only this skill and `weekly-plan`/`roadmap`/`log-feedback` write state.** Confirm before writing.
