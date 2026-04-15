---
description: Capture raw customer feedback into project-state.md § Customer feedback inbox so it is not lost in DMs. Feedback is appended with source, tier, date and quote; it is not turned into Fizzy cards immediately — that happens during /project-manager:groom --ingest-feedback. Keeps the conversation surface low-friction for the founder.
model: sonnet
---

# Log feedback — customer feedback inbox

## Input

`$ARGUMENTS` — the feedback text. Mandatory. May include structured flags:

- `--from=<name-or-handle>` — who said it.
- `--tier=<Basic|Pro|Plus|Prospect|Other>` — customer tier (for prioritization lens, per `domain-heuristics.md § 3`).
- `--channel=<whatsapp|email|call|dm|meeting|other>` — where it came from.
- `--date=<YYYY-MM-DD>` — defaults to today.

If the raw text is missing → abort: `"Usage: /project-manager:log-feedback \"<quote o resumen>\" --from=<name> --tier=<Pro> [--channel=whatsapp]"`.

---

## Step 1 — Load minimal context

Read:
- `${CLAUDE_PLUGIN_ROOT}/context/domain-heuristics.md` — for the tier-prioritization note.

No Fizzy calls. No sprint logic. Keep this skill fast.

---

## Step 2 — Resolve external state

Resolve `project-state.md` path. If missing → stop and ask to run `/project-manager:bootstrap`.

Read the current `Customer feedback inbox` section content.

---

## Step 3 — Parse + normalize

From `$ARGUMENTS`:
- Quote = raw text minus flags, trimmed.
- Source = `<name> (<tier>)` if both provided; else whichever is available; else `"anónimo"`.
- Channel = from flag or `"unspecified"`.
- Date = from flag or today's date (ISO).

If both `--from` and `--tier` are missing, ask once: `"¿De quién vino y en qué tier? (Basic/Pro/Plus/Prospect/Other) — si no sabés, respondé 'anónimo'."` and wait.

---

## Step 4 — Classify tentative action

Using `domain-heuristics.md § 3 & 4`, assign a tentative classification:

| Signal in the quote | Tentative action |
|---|---|
| Bug / broken ("no funciona", "se trabó", "me cobró doble") | `card-probable:type:bug` |
| Feature request ("estaría bueno que…", "me serviría…") | `card-probable:type:feature` |
| Competitive / market info ("el otro servicio hace X") | `note-only:research` |
| Pricing friction ("me parece caro", "no justifica $X") | `note-only:pricing-lens` |
| Praise / signal ("me encantó", "se lo recomendé a Y") | `note-only:retention-signal` |

This is a **suggestion, not a commitment**. No card gets created here.

Bump priority hint if `tier=Pro` or `tier=Plus` per `domain-heuristics.md § 3` (tier matters).

---

## Step 5 — Show the proposed inbox entry

```
### Nueva entrada al feedback inbox

Fecha:        <YYYY-MM-DD>
Fuente:       <name> (<tier>) — canal <channel>
Quote:        "<cleaned quote>"
Tentative:    <card-probable:type:bug | note-only:retention-signal | ...>
Tier weight:  <+0 | +1 — Pro bump | +2 — Plus bump>
```

Ask: `"¿La agrego al inbox? (sí / edita X / no)"` and **wait**.

---

## Step 6 — Append to `project-state.md § Customer feedback inbox`

Format (newest first):

```
- YYYY-MM-DD — <source> — "<quote>" — tentative: <classification> (+tier-weight)
```

Update `Last updated` at the top of the file to today's date and `log-feedback`.

Do **not** modify any other section.

---

## Step 7 — Report

```
### Feedback loggeado

Entrada añadida al inbox (total ahora: <n>).

Siguientes pasos:
  - Si es un bug urgente (tier Pro/Plus que bloquea uso): /project-manager:create-card "<descripción>" --from-customer=<name/tier>.
  - Si no, se procesará durante el próximo /project-manager:groom --ingest-feedback.
```

Optionally, if the entry is `card-probable:type:bug` with `tier=Pro|Plus`, proactively suggest:

```
⚠️ Esto parece bug de cliente de tier <Pro/Plus> — la heurística recomienda priorizarlo.
   ¿Querés que lo convierta en card ya (/project-manager:create-card) o lo dejamos en inbox para el próximo groom?
```

Wait for the answer only if the user engages; otherwise the entry is already in the inbox and the skill can end.

---

## Hard rules

- **Never create a Fizzy card from this skill.** Cards are created by `create-card` (explicit) or `groom --ingest-feedback` (batched).
- **Never touch other sections of `project-state.md`.**
- **Confirm before writing.** The quote is user data — don't paraphrase silently.
- **Keep it fast.** This skill must be usable in 30 seconds so the founder actually runs it instead of letting feedback rot in DMs.
- **Tier matters.** Always surface the tier weight.
