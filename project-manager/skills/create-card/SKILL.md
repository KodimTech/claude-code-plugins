---
description: Create a well-formed Fizzy card on the Kathy board from a raw idea. Builds title, user story, acceptance criteria, tags (type/priority/area/repo/size) and either a technical handoff block (dev cards) or an Outcome esperado block (ops cards). Supports --from-customer to mark retention-signal cards. Inference surfaces uncertainty explicitly — never infers silently.
model: sonnet
---

# Create card — Kathy

## Input

`$ARGUMENTS` — the raw idea / request. Mandatory.

Optional flags embedded in `$ARGUMENTS`:
- `--ops` — force `type:ops` + `repo:none`. For calls, decisions, Meta follow-ups.
- `--from-customer=<tier/name>` — mark the card as triggered by real customer feedback; adds a comment "Origen: feedback de <tier/name>." and nudges priority higher when the tier is Pro ($30) or Plus ($60).
- `--priority=<P0|P1|P2|P3>` — override inferred priority.

If no raw idea → abort: `"Usage: /project-manager:create-card \"<descripción>\" [--ops] [--from-customer=<tier/name>] [--priority=P1]"`.

---

## Step 1 — Load context (parallel)

Read:
- `${CLAUDE_PLUGIN_ROOT}/context/kathy.md`
- `${CLAUDE_PLUGIN_ROOT}/context/fizzy-conventions.md`
- `${CLAUDE_PLUGIN_ROOT}/context/pm-playbook.md`
- `${CLAUDE_PLUGIN_ROOT}/context/domain-heuristics.md`

---

## Step 2 — Read external `project-state.md` (for objective context)

Resolve and read `Current objective` + `Known blockers`. If the state file is missing, proceed but skip objective-tie inference and warn the user at the end: `"project-state.md no existe aún — corré /project-manager:bootstrap para que el PM pueda priorizar contra el objetivo del mes."`.

---

## Step 3 — MCP + board

Resolve Fizzy MCP prefix. `list_boards` → "Kathy", cache `board_id` + column IDs (Backlog, Triage).

---

## Step 4 — Parse raw idea

Extract:
- **Outcome** — what should be different after this ships?
- **Role** — founder / business-owner / end-consumer / Kathy.
- **Area signal** — keywords (whatsapp, menú, reserva, admin, billing, prompt, métricas...).
- **Repo signal** — backend / DB / webhook → api; admin UI / operational → admin; customer self-service → customer; multiple → cross.
- **Ops signal** — "llamar", "hablar con", "decidir", "seguir a Meta", "investigar competidores" → ops.
- **Scope** — single outcome, or bundled?

If bundled: stop. `"Esto parece más de un outcome. ¿Lo parto en <N> cards o lo dejo como una grande (que probablemente haya que splittear)?"`. Wait.

---

## Step 5 — Decide and surface inference

**Never infer silently.** Build an inference block the user can challenge:

```
### Lo que estoy infiriendo

  Tipo:     type:<inferred>     confianza: <alta | media | baja>
  Prioridad: P<inferred>        confianza: <…>
  Área:     area:<inferred>     confianza: <…>
  Repo:     repo:<inferred>     confianza: <…>
  Tamaño:   size:<inferred>     confianza: <…>
  Modo:     <dev | ops>

Si algo te parece mal, decímelo antes de que arme la card.
```

**Defaults:**
- Type: `feature` unless keywords flip it (bug/chore/research/spike/ops).
- Priority: `P2` unless urgency keywords ("urgente", "crítico", "bloquea") raise it. `--from-customer=<Pro|Plus>` bumps to P1 by default.
- Area: inferred from keywords; if multi-area, pick the dominant.
- Repo: inferred; if mixed, `repo:cross`; if ops, `repo:none`.
- Size: XS if trivial; S for a small endpoint/column; M if migration/prompt rewrite/new flow; L → stop and ask to split.

**Confidence tagging:**
- `alta` — the text makes it obvious.
- `media` — inferred from context but could be wrong.
- `baja` — guessing. Force a question to the user.

**For every `baja`, ask.** Batch questions into one message, numbered. Do not ask one at a time.

---

## Step 6 — Apply domain heuristic check

Cross-check against `domain-heuristics.md`:
- Sales-related card? → § 1. Does it optimize conversion-at-close, or funnel top?
- Booking card? → § 2. Does it touch capacity read, confirmation loop, cancellation?
- Support card? → § 3. Is the "who's the customer" clear (owner vs end consumer)?

If the card smells wrong ("optimizing amplification before the leak", "building Plus-only feature with zero Plus customers"), add a note to the draft description under "Notas / suposiciones":

```
⚠️ Heurística de dominio (domain-heuristics.md § X):
<lo que recomienda la heurística y por qué esta card podría violarla>
```

The founder can acknowledge or push back — the card still gets created, but with the flag visible.

---

## Step 7 — Draft the card

Use the appropriate template from `fizzy-conventions.md`:

**Dev card:**

```
Title: <outcome-oriented, ≤80 chars>

### Contexto
<1–3 sentences tying to Current objective or 30-customer lever.>

### Historia de usuario
Como <rol>
quiero <outcome>
para <valor>.

### Criterios de aceptación
- [ ] AC 1
- [ ] AC 2
- [ ] AC 3  (mínimo 2)

### Ejemplos
<Si la idea menciona conversaciones, inputs concretos, o casos puntuales, transcribilos acá.
 Para cards de WhatsApp/prompt/booking/sales, intentar siempre dejar al menos un diálogo de ejemplo
 (Cliente: "..." / Kathy: "..."). Si no aplica, omitir el bloque.>

### Fuera de alcance
<Siempre presente. Listar 1–3 ítems que esta card NO resuelve para que el dev no los meta.
 Si genuinamente no hay nada que aclarar, escribir "nada fuera de los AC".>

### Notas / suposiciones
<Only if raw idea provided them, or a domain-heuristics flag fires.>

### Handoff técnico
Repo: <api | admin | customer | cross>
Siguiente paso: abrir el repo correspondiente y correr
  /api-dev:plan <card-id o pegar descripción>
  (usar /customer-dev:plan para React)
El plan técnico queda como comentario en esta card.

Tags: type:<...>, P<...>, area:<...>, repo:<...>, size:<...>
Column: Triage (or Backlog if needs-input)
```

**Ops card (if `--ops` or inferred ops):**

```
Title: <outcome-oriented, ≤80 chars>

### Contexto
<por qué esto es relevante ahora.>

### Objetivo
<una oración: qué es "hecho" cuando termine.>

### Criterios de aceptación
- [ ] AC 1
- [ ] AC 2

### Fuera de alcance
<Obligatorio. Qué decisiones / acciones quedan fuera de esta card.>

### Outcome esperado
<qué se registra al cerrar: decisión en project-state.md § Recent decisions,
 nota en comentarios, email enviado, llamada hecha, etc.>

Tags: type:ops, P<...>, area:<...>, repo:none, size:<...>
Column: Triage (or Backlog if needs-input)
```

---

## Step 8 — Show the draft + inference recap

Output:

```
### Inferencia (recap)
<el bloque de Step 5 con las correcciones del founder si hubo>

### Draft
<el card completo según Step 7>

### Heurística de dominio
<flag si aplica, o "sin objeciones">

### Origen
<si --from-customer: "feedback de <tier/name>"; si no: "ninguno">
```

Ask: `"¿La creo así, edito, o descarto? (sí / cambia X / no)"` and **wait**.

If the user edits, apply and re-show.

---

## Step 9 — Create in Fizzy

On confirmation:

1. Decide target column: `Triage` if all required tags confident; `Backlog` if `needs-input` is needed.
2. `create_card(board_id, title, description, column_id)`.
3. For each tag: `toggle_card_tag(card_id, tag)`.
4. Optional: convert AC into `create_step` calls only if the founder asked. Default: leave AC inline.
5. Append a comment documenting origin:
   - `--from-customer=<tier/name>` → `"Origen: feedback de <tier/name> (<fecha>)."`.
   - otherwise → `"Card creada por /project-manager:create-card a partir de: <primera línea recortada a 140 chars>."`.
6. If a `domain-heuristics.md` flag fired, append a second comment: `"Heurística: <ref>. Recomendación: <acción>."`.

---

## Step 10 — Report

```
### Card creada

ID: #<id>
Columna: <Triage | Backlog>
Tags: <list>
Modo: <dev | ops>
Origen: <none | feedback de X>

Siguiente paso:
  - Dev card: abrir el repo <repo> y correr /<repo>-dev:plan.
  - Ops card: agendar la acción; al cerrarla, registrar el outcome esperado en project-state.md § Recent decisions vía /project-manager:weekly-review.
  - Urgente: /project-manager:weekly-plan para incluirla en el sprint actual.
```

---

## Hard rules

- **Never create a card with missing required tags silently.** Use `needs-input` + leave in Backlog.
- **Never infer silently.** Always show the inference block with confidence levels.
- **AC ≥ 2.** "Funciona" is not an AC.
- **"Fuera de alcance" siempre presente.** Aunque sea para escribir "nada fuera de los AC". El dev debe poder confiar en que lo no listado no se va a pedir después.
- **"Ejemplos" siempre que aplique.** Para cards de conversación, prompt, bug, o UI: pegar al menos un caso concreto. Sin ejemplo, los AC son interpretables.
- **Title names the outcome**, not the implementation step.
- **Ops cards skip Handoff técnico** but keep Outcome esperado. Never fake a `repo:*` on an ops card — use `repo:none`.
- **Do not read source code.**
- **One card per outcome.**
- **Cross-check domain heuristics** and flag violations visibly in the card itself.
