---
description: Backlog grooming for Kathy — includes triage. Classifies untagged cards, dedupes, resizes, re-ranks, flags blockers, splits L cards, and ingests entries from the customer feedback inbox in project-state.md. Supports a --triage-only mode for quick classification. Proposes a diff and applies only after confirmation; low-risk mutations (missing tags, promote to Triage) can be batched optimistically.
model: opus
---

# Groom — Kathy backlog (triage included)

## Input

Optional `$ARGUMENTS`:

- `--triage-only` — only classify untagged Backlog cards and promote the ready ones; skip dedup / split / resize.
- `--scope=<filter>` — narrow by tag (`area:admin-ui`, `P2`, `blocked`) or card age (`older-than=6w`).
- `--ingest-feedback` — first, process pending entries in `project-state.md § Customer feedback inbox` into cards (or discard with the founder), then continue with the normal groom pass.

If no arguments → full groom of Backlog + Triage.

---

## Step 1 — Load context (parallel)

Read:
- `${CLAUDE_PLUGIN_ROOT}/context/kathy.md`
- `${CLAUDE_PLUGIN_ROOT}/context/fizzy-conventions.md`
- `${CLAUDE_PLUGIN_ROOT}/context/pm-playbook.md`
- `${CLAUDE_PLUGIN_ROOT}/context/domain-heuristics.md`

---

## Step 2 — Read external `project-state.md`

Resolve path (env var or `~/.kathy-pm/project-state.md`). If missing → stop with the bootstrap hint.

Extract:
- `Current objective`
- `Known blockers`
- `Customer feedback inbox` — only if `--ingest-feedback` or full groom.

---

## Step 3 — MCP + board

Resolve Fizzy MCP prefix (`1mcp` first, `claude_ai_1MCP` fallback).
`list_boards` → "Kathy", cache ids.

---

## Step 4 — Optional: ingest feedback inbox

Only if `--ingest-feedback` or if the inbox has unprocessed entries during a full groom:

For each entry `YYYY-MM-DD — source (tier/name) — quote — tentative card?`:

1. Show it to the founder.
2. Ask: `"¿Lo convertimos en card, lo agrupo con otro feedback similar, o lo descarto? (card / agrupar / descartar)"`.
3. On `card`: collect the minimum to call `/project-manager:create-card` semantics inline (title, AC draft, tags). Create it. Add comment `"Origen: feedback de <source> (<tier>) del <date>."`.
4. On `descartar`: leave a short reason.
5. Remove the processed entry from the inbox in memory (will be written back in Step 9).

---

## Step 5 — Pull cards

Depending on scope:

- Full: `list_column_cards(board_id, "Backlog")` + `list_column_cards(board_id, "Triage")`.
- `--triage-only`: just `Backlog`, only cards missing any required tag.
- `--scope=<filter>`: apply filter after pulling.

For each card, `get_card` for tags, description, comments, age.

---

## Step 6 — Classify each card

For every card, decide one of:

| Action            | When |
|-------------------|------|
| **keep**          | Fine as is. |
| **retag**         | Missing required tag (type/P*/area/repo/size); infer from text. |
| **promote**       | Fully tagged, description sections present → move Backlog → Triage. |
| **needs-input**   | Description too thin to infer safely → add tag + comment with question. |
| **resize**        | Size tag doesn't match scope drift. |
| **reprioritize**  | Priority no longer ties to current objective. |
| **split**         | `size:L` or card bundles multiple outcomes. |
| **dedupe**        | Similar card exists → close the weaker one with a link (needs confirm). |
| **stale**         | >6 weeks old, no activity, no objective tie → suggest `not-now` or `close` (needs confirm). |
| **unblock-check** | `blocked` tag present; check the reason, nudge if stale. |
| **heuristic-flag**| Card violates a `domain-heuristics.md` rule → annotate + propose defer. |

Detect duplicates by title+description similarity (normalized keywords).
Detect stale via Fizzy's `created_at` / `updated_at`.
Detect heuristic violations by matching the card's `area:*` against the relevant section of `domain-heuristics.md`.

---

## Step 7 — Group mutations by risk

Bucket the proposals:

- **Low-risk (batch apply on single "sí")**:
  - `retag` (missing tag inference)
  - `promote` (Backlog → Triage on complete cards)
  - `unblock-check` (nudge comment only)
  - Adding `needs-input` tag + comment asking the question

- **High-risk (always per-card confirmation)**:
  - `dedupe` (closing something)
  - `stale → not-now / close`
  - `split` (creates new cards + closes original)
  - `resize` / `reprioritize` where the change is non-obvious

---

## Step 8 — Show the grooming diff

```
### Grooming propuesto — <N cards revisadas>

Feedback inbox ingested: <n cards / n discarded / n grouped>   (if applicable)

Low-risk (propuestas a aplicar en bloque):
  | Card | Acción | Cambio |
  |------|--------|--------|
  | #123 | retag  | + type:bug, + area:booking-flow, + repo:api |
  | #145 | promote | Backlog → Triage |
  | #156 | needs-input | + tag + comentario "¿cuál es el AC 2?" |

High-risk (revisemos uno a uno):
  | Card | Acción | Razón |
  |------|--------|-------|
  | #160 | split | size:L — propuesta de corte: (a) read menu | (b) edit menu |
  | #172 | dedupe de #98 | títulos ~90% overlap |
  | #180 | not-now | 8 semanas sin tocar, tema deprecado |
  | #190 | heuristic-flag | Choca con domain-heuristics § 1 (funnel top > conversion); recomiendo bajar a P3 |

Sin cambios: <n>
```

---

## Step 9 — Confirm

Two confirmations:

1. **Low-risk batch**: `"¿Aplico todos los cambios low-risk? (sí / no)"` → apply as one sweep on "sí". Apply nothing on "no".
2. **High-risk iteration**: for each high-risk row, show the proposed action and ask `"¿Aplico? (sí / cambiá X / skip)"`. Iterate.

If the user wants full interactive from the start, they can pass no `$ARGUMENTS` and say "revisemos uno a uno" — then everything goes per-card.

---

## Step 10 — Apply changes

Per approved change:

- **retag** → `toggle_card_tag` per tag. Comment: `"Triaged/regroomed: tags añadidos <list>."`.
- **promote** → `update_card(id, triage_column_id)` + comment `"Promovida a Triage: tags completos, descripción OK."`.
- **needs-input** → `toggle_card_tag(id, "needs-input")` + specific question comment.
- **resize / reprioritize** → toggle old off + new on + reason comment.
- **split** → `create_card` per new slice with minimal description; `close_card(original)` with links comment.
- **dedupe** → `close_card(weaker_id)` + comment linking the surviving card.
- **stale → not-now** → `mark_card_not_now(id)` + reason comment.
- **stale → close** → `close_card(id)` + reason comment.
- **unblock-check** → comment only, nudging the founder.
- **heuristic-flag** → toggle priority change (if approved) + comment citing the exact `domain-heuristics.md §`.

Never delete. Never overwrite a comment.

---

## Step 11 — Update external state (only feedback inbox)

If feedback inbox entries were processed in Step 4, write the updated inbox (without processed entries) back to the external `project-state.md`. Update `Last updated`.

Do **not** touch other sections — those belong to `weekly-review`, `weekly-plan`, and `roadmap`.

---

## Step 12 — Report

```
### Grooming aplicado

Low-risk aplicados: <n>  (retag/promote/needs-input/unblock-check)
High-risk aplicados: <n>  (split/dedupe/not-now/close/resize/reprioritize)
Heuristic flags: <n>     (referencias citadas en los comentarios)
Feedback inbox procesado: <n> → <m cards, k descartadas, j agrupadas>

Triage queda con: <n> cards, <m> READY por Definition of Ready.
Backlog queda con: <n> cards, <m> con needs-input.

Próximos pasos sugeridos:
  - /project-manager:weekly-plan   — si Triage tiene cards ready suficientes
  - Revisar cards con needs-input: <ids>
```

---

## Hard rules

- **Always show the diff before writing.**
- **Never delete cards.** Close or not-now.
- **Splits must be independent outcomes**, not layer splits.
- **Every mutation gets a comment** naming the reason. For heuristic flags, cite the exact `domain-heuristics.md §`.
- **Dedup decisions need human approval.**
- **Stale ≠ dead.** Confirm before closing an old card.
- **Low-risk vs high-risk** buckets are non-negotiable — never auto-close in the low-risk batch.
