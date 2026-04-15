# Fizzy board conventions — Kathy

This file is the contract between the PM agent and the Fizzy board. Every skill relies on it. If the board drifts from this file, either fix the board (via `bootstrap`) or update this file — never let them disagree.

---

## Board

- **Name:** `Kathy`
- **One board, one product.** No parallel boards per repo; repo affinity is expressed via tag (`repo:*`).
- Discover the board ID at the start of each session with `list_boards` and cache it in memory.

---

## Columns (the flow)

Left to right:

1. **Backlog** — every incoming idea/request/bug lands here. Not prioritized. May be untagged.
2. **Triage** — reviewed once: has tags, description, size. Ready to be considered for a sprint.
3. **This Week** — committed for the active sprint (has a `sprint:YYYY-Www` tag). Upper bound: 10h of estimated work.
4. **In Progress** — actively being worked on. Target WIP = 1. Max = 2 (solo dev; more than 2 is thrashing).
5. **Review / Testing** — code done, self-review / manual testing / PR in review. Waiting on verification.
6. **Done** — shipped to main / production. Closed cards live here until archived at the end of the sprint.

Additional states handled via Fizzy's built-in mechanisms (not columns):
- **Not now** — `mark_card_not_now` for cards we've decided to defer without closing.
- **Closed** — `close_card` when finished or abandoned (with a comment explaining why if abandoned).

---

## Tags (the metadata layer)

Fizzy only supports flat tags, so the schema is prefixed.

### Type

- `type:feature` — new user-visible capability
- `type:bug` — something is broken
- `type:chore` — infra / cleanup / deps / docs
- `type:research` — time-boxed investigation, no commitment to ship
- `type:spike` — prototype to answer a specific question, code is throwaway
- `type:ops` — **no code** involved: calls, decisions, Meta follow-ups, pricing workshops, external comms. See `pm-playbook.md § Operational cards`.

### Priority

- `P0-critical` — production is broken or a customer promise is about to break; drops everything
- `P1-high` — this sprint
- `P2-medium` — next 2-3 sprints
- `P3-low` — nice to have, no commitment

### Area (functional slice of Kathy)

- `area:whatsapp` — WhatsApp integration, webhooks, Meta permissions, message delivery
- `area:sales-flow` — conversion / pre-sale conversations
- `area:booking-flow` — reservations, scheduling, calendar
- `area:menu-catalog` — products, services, menu management
- `area:prompt` — AI prompt quality, personality, edge cases
- `area:admin-ui` — business-owner operational panel
- `area:customer-ui` — self-service / billing / account panel
- `area:billing` — subscriptions, tiers, payment
- `area:infra` — deployment, background jobs, observability, cost control
- `area:analytics` — metrics, dashboards, conversion tracking

### Repo (who implements)

- `repo:api` — Rails
- `repo:admin` — React admin panel
- `repo:customer` — React customer panel
- `repo:cross` — touches more than one codebase
- `repo:none` — no code involved (ops/decision/call/research). Must be paired with `type:ops` or `type:research`.

### Size (rough estimate, not story points)

- `size:XS` — ≤1h     (capacity midpoint **0.75h**)
- `size:S`  — 1–3h    (capacity midpoint **2h**)
- `size:M`  — 3–6h    (capacity midpoint **4.5h**)
- `size:L`  — >6h     → **must be split before entering a sprint**

The midpoints above are what `weekly-plan` and `weekly-review` sum for capacity math. Consistent numbers everywhere.

### Sprint

- `sprint:YYYY-Www` (ISO week, e.g., `sprint:2026-W16`). Applied when a card is moved into `This Week`.

### Optional

- `blocked` — dependency outside the dev's control (e.g., Meta permission, third-party API). Pair with a comment naming the blocker.
- `needs-input` — waiting on the founder to answer a question. Resolve before a sprint start.

---

## Card shape (what a well-formed card looks like)

Every card's description follows this template:

```markdown
### Contexto
<1–3 sentences: what's the user-visible problem, why it matters for Kathy's 30-customer goal or MVP blockers.>

### Historia de usuario
Como <rol: founder | business-owner | end-consumer | Kathy-itself>
quiero <resultado observable>
para <valor concreto>.

### Criterios de aceptación
- [ ] AC 1
- [ ] AC 2
- [ ] AC 3

### Ejemplos
<Opcional pero muy recomendado cuando aplica. Casos concretos de input → output esperado.
 Para Kathy, lo natural es pegar conversaciones de WhatsApp:
   Cliente: "Hay mesa para 4 esta noche?"
   Kathy:   "Sí, tengo a las 20:30 o 21:45. ¿Cuál preferís?"
 También sirve para bugs ("antes esto pasaba X, ahora pasa Y"), edge cases, o estados de UI.>

### Fuera de alcance
<Obligatorio. Lo que esta card explícitamente NO incluye, para evitar scope creep durante el dev.
 Ejemplos: "no incluye notificación al dueño", "no soporta modificadores del menú",
           "no toca el flujo de cancelación".
 Si todo está en alcance, escribir "nada — el alcance es el de los AC".>

### Notas / suposiciones
<Anything the dev needs to know that isn't in the AC: assumptions, open questions, UX sketches, links.>

### Handoff técnico
Repo: <api | admin | customer | cross>
Siguiente paso: dentro del repo correspondiente, correr
  /api-dev:plan <card-id o pegar esta descripción>
  (o /customer-dev:plan para React)
El plan técnico debe quedar como comentario en esta card.
```

If any section is empty, the card is not ready — goes to `Triage` with `needs-input`.

### Ops card variant (`type:ops` + `repo:none`)

Ops cards replace "Handoff técnico" with a lighter "Outcome esperado" section, since no dev is picking this up:

```markdown
### Contexto
<por qué esto es relevante ahora.>

### Objetivo
<una oración: qué se considera "hecho" cuando termine este trabajo.>

### Criterios de aceptación
- [ ] AC 1
- [ ] AC 2

### Fuera de alcance
<Obligatorio. Qué decisiones / acciones quedan fuera de esta card.
 P.ej.: "no se decide pricing acá, solo se valida el flujo de aprobación de Meta".>

### Outcome esperado
<qué se registra al cerrar: decisión en `project-state.md § Recent decisions`,
 nota en comentarios, email enviado, etc.>
```

Everything else (tags, column flow) is identical to a dev card.

### Steps (Fizzy's subtask mechanism)

Use `create_step` for Acceptance Criteria when it helps visibility. Otherwise keep AC inline in the description. Do not duplicate.

### Tags required at minimum

On every card before leaving `Triage`:

- one `type:*` (feature / bug / chore / research / spike / ops)
- one `P0/P1/P2/P3`
- one `area:*`
- one `repo:*` (use `repo:none` for `type:ops`)
- one `size:*` (XS / S / M; L is not allowed in a sprint)

Missing any → add `needs-input` and leave in Triage.

---

## Sprint lifecycle

1. **Monday (weekly-plan)** — compute sprint tag `sprint:YYYY-Www`, pick cards from Triage adding up to ≤10h, move them to `This Week` with the sprint tag applied.
2. **Mid-week (status on demand)** — summarize what's in `In Progress`, flag blockers.
3. **Friday (weekly-review)** — list what closed this sprint, what didn't and why, update `project-state.md`, leave unfinished cards in `This Week` or demote to `Triage` depending on call.

---

## Conventions the PM respects

- **Never delete a card.** Use `close_card` or `mark_card_not_now`, with a comment explaining why.
- **Never rewrite someone else's comment.** Append a new one.
- **Always comment when moving a card backwards** (e.g., `In Progress` → `Triage`). State the reason.
- **When in doubt, use `triage_card`** to send something to Triage rather than assuming tags.
