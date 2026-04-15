# Project state — Kathy

> **This is a template.** The real project-state lives **outside the plugin directory** so it survives `claude plugin update`. Default path:
> `~/.kathy-pm/project-state.md`
> Override with env var `KATHY_PM_STATE_PATH` if you prefer another location (e.g., inside a private repo you version-control).
>
> `/project-manager:bootstrap` copies this template to the real path on first run. After that, `weekly-plan`, `weekly-review`, `roadmap` and `log-feedback` read and write the **real** file, never this template.

---

## Last updated

`YYYY-MM-DD` by `weekly-review | weekly-plan | roadmap | log-feedback | manual edit`

---

## Current objective (this month)

<!-- One sentence. The single most important outcome the founder is pushing toward right now. -->

_Not yet set. Run `/project-manager:roadmap` to define._

---

## Active sprint

- **Tag:** `sprint:YYYY-Www`
- **Window:** `YYYY-MM-DD → YYYY-MM-DD`
- **Planned:** 0h
- **Status:** not started

_Populated by `/project-manager:weekly-plan`._

---

## Rolling capacity

> **Calibrated** by `weekly-review` from the last 3 closed sprints. Start at 10h. If the founder consistently closes less than planned, the plan shrinks automatically so it stops lying to itself.

- **Current capacity:** `10h` (seed value — not yet calibrated)
- **Rolling sample (last 3 sprints):** _none yet_
  - `sprint:YYYY-Www` — planned `Xh` / closed `Yh`
  - ...
- **Calibration rule:** after 3 sprints, set `Current capacity = round(average(closed) + 1h buffer)`, clamped between 4h and 10h.

---

## Known blockers

<!-- External things the founder can't just code around. Update when a new one appears or an old one resolves. -->

- **Meta WhatsApp Business Management permission** — pending approval. Blocks production onboarding at scale. Any card that needs production WhatsApp at real volume is gated by this.

---

## Recent decisions (last 4 weeks)

<!-- Append new decisions at the top. Drop entries older than ~4 weeks unless they're load-bearing. Format: `YYYY-MM-DD — decision — why`. -->

_Empty._

---

## Customer feedback inbox

<!-- Populated by `/project-manager:log-feedback`. Raw inputs from real users before they become cards. Format: `YYYY-MM-DD — source (tier/name) — quote — tentative card?`. Cleared by `/project-manager:groom` when entries turn into cards or are discarded. -->

_Empty._

---

## Retro — last sprint

<!-- Populated by `/project-manager:weekly-review`. Three columns: what went well / what didn't / what to change. -->

_No sprint closed yet._

---

## Roadmap snapshot (next 1–3 months)

<!-- High-level. Not cards, not dates. Populated by `/project-manager:roadmap`. Each entry is a theme with a target month. -->

_Not yet set._

---

## Open questions for the founder

<!-- Things the PM is unsure about and needs clarity on. Clear these during `weekly-plan` or `weekly-review`. -->

_None._
