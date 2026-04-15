---
description: Plan the next 1–3 months for Kathy. Reads external project-state.md and kathy.md, asks the founder for the next outcome, maps it to themes & epics (outcome-level), cross-checks against domain-heuristics (tier lens, 30-customer filter, WhatsApp gravity). Writes a Roadmap snapshot into project-state.md after confirmation.
model: opus
---

# Roadmap — Kathy 1–3 month plan

## Input

Optional: `$ARGUMENTS` may contain a seed theme.

---

## Step 1 — Load context (parallel)

Read:
- `${CLAUDE_PLUGIN_ROOT}/context/kathy.md`
- `${CLAUDE_PLUGIN_ROOT}/context/pm-playbook.md`
- `${CLAUDE_PLUGIN_ROOT}/context/domain-heuristics.md`

---

## Step 2 — Read external `project-state.md`

Resolve + read:
- `Current objective`
- `Recent decisions`
- `Known blockers`
- `Roadmap snapshot` (prior, if any)
- `Retro — last sprint`
- `Customer feedback inbox` — as signal for what's coming up from real users.

If state file missing → stop, hint at `/project-manager:bootstrap`.

---

## Step 3 — Survey recent activity

Resolve Fizzy MCP + `list_boards`.

In parallel:
- Cards in `Done` across the last 3 sprints (filter by `sprint:*` tags older than current).
- Top 20 cards in `Triage` + `Backlog` by priority.
- Cards tagged `blocked`.

Build a signal summary: where time went, what's piling up, what's structurally stuck.

---

## Step 4 — Ask the founder

```
### Roadmap — contexto actual

Objetivo vigente (project-state.md):
  <...>

Últimos 3 sprints (distribución):
  Área: ...
  Repo: ...
  Ops:  <n>h  (si hubo)

Top Triage/Backlog por prioridad:
  - <...>

Bloqueos estructurales:
  - <p.ej. "Meta permission — pending">

Señales del feedback inbox (últimas <n> entradas):
  - <resumen agregado>

Para armar el roadmap necesito saber:
  1. ¿Cuál es el outcome principal de los próximos 1–3 meses?
     (no listes features — describí el resultado: p.ej. "10 clientes en tier Pro",
      "cualquier restaurante puede auto-servirse el menú", "Kathy maneja menús con modificadores sin fallar")
  2. ¿Hay deadlines externos? (feria, demo, charla, compromiso comercial)
  3. ¿Hay algo que explícitamente NO va a pasar en estos 3 meses?
  4. ¿Hay tier específico en el que querés invertir? (domain-heuristics § 4)
```

**Wait** for the answer.

---

## Step 5 — Propose themes + epics

From the founder's answer, build 2–4 **themes** (outcome-level). For each:
- 2–4 **epics** (still outcome-level, not feature lists).
- Target month.
- Dependencies / risks.

Cross-check every theme and epic against:

- `domain-heuristics.md § 4 — Pricing tier lens`: is this building for a tier where you already have customers?
- `domain-heuristics.md § 4 — 30-customer heuristic`: does this epic realistically convert a prospect into a paying customer?
- `domain-heuristics.md § 4 — WhatsApp gravity`: is the theme's delivery gated by Meta?
- `domain-heuristics.md § 4 — Cost of conversation`: does this epic raise LLM cost per conversation without clear payback?

Flag any theme that violates a heuristic — don't silently accept it.

---

## Step 6 — Draft the snapshot

```
### Roadmap propuesto — <start-month> a <end-month>

Outcome (una oración): <tomado de la respuesta del founder>

| Tema | Epics (outcome-level) | Target | Depende de |
|------|-----------------------|--------|-----------|
| <tema 1> | - <epic>  - <epic>  - <epic> | <mes> | <blocker / none> |
| <tema 2> | - <epic>  - <epic>            | <mes> | <...> |
| <tema 3> | - <epic>  - <epic>            | <mes> | <...> |

Heurísticas de dominio (cross-check):
  - ✅ Tema <1> empuja tier Pro, que es donde hay tracción → alineado con § 4.
  - ⚠️ Tema <2> depende de Meta permission — marcado como riesgo.
  - ❌ Epic "<X>" choca con § 1 (funnel top > conversion) → ¿bajamos prioridad o lo movemos a un trimestre posterior?

Lo que explícitamente NO vamos a hacer:
  - <list>

Riesgos / dependencias externas:
  - <...>

Lectura del roadmap:
  - <qué implica para el día a día (1 oración)>
  - <qué cambia en la priorización del backlog (1 oración)>
  - <cambio propuesto al Current objective (1 oración)>
```

---

## Step 7 — Confirm

Ask: `"¿Guardo este roadmap en project-state.md y ajusto el Current objective? (sí / cambiá X / no, solo era exploración)"` and **wait**.

Iterate on edits.

---

## Step 8 — Update external `project-state.md` (show diff first)

Proposed edits:

1. `Last updated` → today + `roadmap`.
2. `Current objective` → one-sentence outcome from Step 5.
3. `Known blockers` → add any new external deps surfaced during the conversation.
4. `Roadmap snapshot` → replace with the new table.
5. `Open questions for the founder` → any unresolved item.

Show the diff. Ask one more `"¿Aplico?"`. On "sí" → `Edit` the external file.

---

## Step 9 — Follow-up suggestions

```
Siguiente paso sugerido:
  - /project-manager:groom      — realinear backlog con el nuevo roadmap
  - /project-manager:weekly-plan — si querés que el sprint de esta semana ya empuje el tema #1

Nota:
  No creo cards de epics automáticamente. Un epic es un outcome; se descompone en cards concretas
  durante /project-manager:groom o /project-manager:create-card, no acá.
```

---

## Hard rules

- **Outcomes, not features.** Epic = user-observable state change.
- **Never write file-level or API-level detail.**
- **Always ask the founder for the outcome.** No inventing direction.
- **Cross-check against domain heuristics** and flag violations explicitly.
- **Dependencies = risks**, not epics.
- **Months only**, no dates.
- **Update external `project-state.md`**, never the template.
- **Confirm twice** — once on the proposal, once on the state-file diff.
