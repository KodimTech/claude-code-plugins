# project-manager

Senior Project Manager assistant for the **Kathy** product — an AI agent that answers WhatsApp messages for service businesses so they stop losing sales and bookings.

The plugin runs a weekly PM cadence against a single **Fizzy** board (also called `Kathy`) for a solo dev on a **calibrated weekly capacity** (seeded at 10h, shrinks as the Rolling capacity reflects reality). It plans, grooms, triages, captures customer feedback, and retrospects — but it does **not** read source code. Technical breakdown of each card is done later inside the relevant repo using `/api-dev:plan` or `/customer-dev:plan`.

Version **0.2.1** — this README describes the reworked plugin after the post-v0.1 self-review. Changes from v0.1 are listed at the bottom.

---

## Requirements

- **Claude Code** with the plugin marketplace `kodim-api` installed.
- **Fizzy MCP** connected (through `1mcp` or `claude.ai 1MCP`) exposing the `Fizzy_1mcp_*` tools. Both prefixes are supported — the plugin resolves to whichever is available:
  - `mcp__1mcp__Fizzy_1mcp_*`
  - `mcp__claude_ai_1MCP__Fizzy_1mcp_*`
- A Fizzy board named exactly **`Kathy`**.
- An external state file at `~/.kathy-pm/project-state.md` — created by `bootstrap`. Override the location with `KATHY_PM_STATE_PATH`.

No local tooling beyond Claude Code.

---

## Install

From any Claude Code session:

```
/plugin marketplace add KodimTech/claude-code-plugins
/plugin install project-manager@kodim-api
```

Then run **once**:

```
/project-manager:bootstrap
```

It will:

1. Create `~/.kathy-pm/project-state.md` (or `$KATHY_PM_STATE_PATH`) from the template shipped in the plugin. **This file is your long-term memory and lives outside the plugin, so it survives `claude plugin update`.**
2. Ensure the `Kathy` board has the canonical columns and tag taxonomy from `context/fizzy-conventions.md`.

Idempotent — safe to re-run.

---

## Skills

| Skill | Cadence | Model | What it does |
|---|---|---|---|
| `/project-manager:bootstrap` | one-time | Sonnet | Initialize external state file + ensure board columns/tags. |
| `/project-manager:weekly-plan` | Monday | Opus | Build the week's sprint using **calibrated capacity** (not a hard 10h), narrative per-card verdicts (no numeric score), tag with `sprint:YYYY-Www`. |
| `/project-manager:groom` | weekly / on-demand | Opus | Dedupe, close stale, resize, re-rank, flag blockers, promote tagged cards to Triage. **Includes triage** (no separate skill). `--triage-only` mode for quick classification. `--ingest-feedback` processes feedback inbox. Low-risk mutations batch-apply; high-risk stay per-card. |
| `/project-manager:create-card "<idea>"` | on-demand | Sonnet | Build a well-formed card (story + AC + tags + handoff). Supports `--ops` and `--from-customer=<tier/name>`. Shows inference with confidence levels — never silent. |
| `/project-manager:log-feedback "<quote>" --from=X --tier=Pro` | on-demand | Sonnet | Capture raw customer feedback into `project-state.md § Customer feedback inbox`. Does not create Fizzy cards; those come later via `groom --ingest-feedback`. |
| `/project-manager:status` | mid-week | Sonnet | Read-only snapshot of the active sprint. |
| `/project-manager:weekly-review` | Friday | Opus | Close the sprint, retro, update `project-state.md`, **calibrate Rolling capacity** from planned-vs-closed. |
| `/project-manager:roadmap` | monthly | Opus | Map next 1–3 months into themes & epics, cross-check domain heuristics, write snapshot to `project-state.md`. |

No `ship` orchestrator — skills are invoked at the right moment of the week.

---

## Capacity calibration — the honesty loop

The plugin does **not** assume a hard 10h/week. It maintains a Rolling capacity in `project-state.md`:

- Seed: 10h.
- After every `weekly-review`: appends `sprint:YYYY-Www — planned Xh / closed Yh` to the sample.
- Once the sample has ≥ 3 entries: recomputes `Current capacity = round(avg(closed) + 1h buffer)`, clamped `[4h, 10h]`.
- `weekly-plan` plans to `Current capacity`, not to 10.

If you close 6h for three sprints straight, the plan shrinks to 7h. No more fiction.

**Overrides:** pass `capacity=4` to `/project-manager:weekly-plan` for a one-off (holiday week, burst week) without changing the rolling value.

---

## Domain heuristics — why this PM is not generic

`context/domain-heuristics.md` encodes experience in three domains Kathy cares about:

- **Sales** — conversion-at-close matters more than funnel top; menu-as-catalog beats menu-as-doc; kill silent failures.
- **Reservations** — real-time capacity read before pretty UI; confirmation reliability before speed; cancellation flow is not optional; reminders reduce no-shows 20–40%.
- **Customer service** — two customers (business owner vs end consumer) with different SLAs; detection beats resolution; tier matters (Pro/Plus complaints are upgrade signals); trust is asymmetric.
- **Cross-cutting** — pricing tier lens, 30-customer filter, cost of conversation, WhatsApp gravity (Meta approval, template vs session messages).

Every card creation and grooming decision cross-checks against this file and flags violations explicitly. Example flag on a card:

> ⚠️ Heurística de dominio (domain-heuristics.md § 1): optimizing funnel top while conversion-at-close is still shaky. Recomiendo deprior hasta tener datos base.

---

## Memory — how the PM stays coherent across sessions

`~/.kathy-pm/project-state.md` (external — survives `plugin update`):

- **Last updated** — date + skill that wrote it.
- **Current objective** — the one outcome driving this month.
- **Active sprint** — tag, window, planned, status.
- **Rolling capacity** — calibrated budget + sample of last 3 sprints.
- **Known blockers** — external deps (Meta, payments, etc.).
- **Recent decisions** — last 4 weeks, newest first.
- **Customer feedback inbox** — raw inputs from real users (via `log-feedback`), pending card conversion.
- **Retro — last sprint** — 3 columns (went well / didn't / change).
- **Roadmap snapshot** — 1–3 month themes & epics.
- **Open questions for the founder**.

Only `weekly-plan`, `weekly-review`, `roadmap`, and `log-feedback` write to this file — always after explicit confirmation.

---

## Board conventions (quick reference)

Full spec in `context/fizzy-conventions.md`.

**Columns**: `Backlog → Triage → This Week → In Progress → Review / Testing → Done`

**Required tags on every ready card**:

- `type:*` — feature / bug / chore / research / spike / **ops**
- `P0-critical` · `P1-high` · `P2-medium` · `P3-low`
- `area:*` — whatsapp / sales-flow / booking-flow / menu-catalog / prompt / admin-ui / customer-ui / billing / infra / analytics
- `repo:*` — api / admin / customer / cross / **none** (ops)
- `size:*` — XS (≤1h, midpoint 0.75) · S (1–3h, midpoint 2) · M (3–6h, midpoint 4.5) · L (>6h, must be split)

**Sprint tag**: `sprint:YYYY-Www` (ISO week). Applied by `weekly-plan`.

**Dev card template**: Contexto · Historia · AC · Ejemplos · Fuera de alcance · Notas · Handoff técnico
**Ops card template**: Contexto · Objetivo · AC · Fuera de alcance · Outcome esperado

Midpoints are used everywhere (plan / status / review) for consistent capacity math.

---

## Handoff with `api-dev` and `customer-dev`

The PM does **not** read source code. Every dev card includes a **Handoff técnico** section:

```
Repo: admin
Siguiente paso: dentro de core-web-customer, correr
  /customer-dev:plan <card-id o pegar descripción>
El plan técnico queda como comentario en esta card.
```

Ops cards (`type:ops` + `repo:none`) skip this — they have an **Outcome esperado** block instead.

---

## Typical week

```
Monday   → /project-manager:weekly-plan
Tue-Thu  → write code (inside core-web-api / admin / customer, using /api-dev or /customer-dev)
           /project-manager:status                 mid-week pulse
           /project-manager:create-card "..."      new idea → card
           /project-manager:log-feedback "..."     customer feedback landed in DMs
Friday   → /project-manager:weekly-review          closes sprint + calibrates capacity
Monthly  → /project-manager:roadmap

As needed:
  /project-manager:groom                     reorganize / dedupe
  /project-manager:groom --triage-only       quick classify of incoming Backlog
  /project-manager:groom --ingest-feedback   turn feedback inbox into cards
```

---

## What this plugin deliberately does NOT do

- Does **not** read code, run tests, open PRs, or invoke git.
- Does **not** auto-create sprint cards — every plan is proposed and confirmed first.
- Does **not** invent dates.
- Does **not** break cards into files, services, endpoints, or components.
- Does **not** run without the Fizzy MCP — stops with a clear error.
- Does **not** plan above `Rolling capacity` without an explicit override.
- Does **not** silently infer tags — always surfaces confidence and asks when low.

---

## Changes in 0.2.1 (vs 0.2.0)

1. **Card template enriched for the dev who actually has to build it.** Two new sections:
   - **Ejemplos** (optional but pushed for conversation/prompt/UI/bug cards): concrete input/output samples — for Kathy this almost always means a WhatsApp dialogue snippet.
   - **Fuera de alcance** (always present): the 1–3 things this card explicitly does NOT solve, so scope creep is caught at planning time, not mid-implementation.
2. `create-card` now fills both sections automatically when context allows, and refuses to silently leave "Fuera de alcance" empty.

## Changes in 0.2.0 (vs 0.1.0)

From the self-review:

1. **External state.** `project-state.md` moved out of the plugin to `~/.kathy-pm/project-state.md` (or `$KATHY_PM_STATE_PATH`). It used to live inside the plugin, which meant `claude plugin update` could wipe it.
2. **Domain heuristics.** Added `context/domain-heuristics.md` covering sales / reservations / customer service, cross-checked in every skill. The v0.1 agent was a generic PM; this one has opinions.
3. **Consistent size → hours.** Explicit midpoints (XS=0.75, S=2, M=4.5, L=excluded). Used identically across plan / status / review.
4. **No numeric score.** `weekly-plan` now tells a 3-line story per card (Value / Cost / Risk / Verdict) instead of a pseudo-precise arithmetic score.
5. **Capacity calibration loop.** `weekly-review` appends planned-vs-closed to `Rolling capacity`; after 3 sprints the budget recomputes. `weekly-plan` reads it.
6. **Ops cards.** `type:ops` + `repo:none` for non-code work (calls, decisions, Meta follow-ups). Ops cards skip "Handoff técnico" and use "Outcome esperado" instead.
7. **Customer feedback intake.** New `log-feedback` skill keeps a feedback inbox in `project-state.md`. `groom --ingest-feedback` promotes entries into cards later.
8. **Triage merged into groom.** The old `triage` skill was folded into `/project-manager:groom`. Use `--triage-only` for a quick pass.
9. **Explicit MCP prefix resolution.** Every skill documents the `1mcp` → `claude_ai_1MCP` fallback and stops with a clear message if neither is connected.
10. **Softer confirmation friction.** `groom` splits mutations into low-risk (batch apply on one "sí") and high-risk (per-card confirmation). `create-card` shows an inference block with confidence levels instead of asking one field at a time.
11. **No more carried-over bonus.** Cards that keep slipping are flagged for questioning, not promoted.

---

## Updating the plugin

After a change, bump `version` in `.claude-plugin/plugin.json` so `claude plugin update` picks it up. **Your `project-state.md` is external — it does not move with the plugin.**
