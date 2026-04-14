---
description: "End-to-end orchestrator for a Notion task in core-web-customer: runs customer-dev:plan → human checkpoint → customer-dev:execute → customer-dev:review → human checkpoint → commit/push/PR → customer-dev:pr-ready. Two explicit checkpoints (plan approval, commit approval); everything else is automated."
model: opus
---

# Ship — Full-flow orchestrator

Chains the `customer-dev` skills in sequence with **two explicit human checkpoints**. The goal: ship a Notion task end-to-end with minimal friction, while keeping the human in control of the two decisions that matter (is the plan good, and is the code ready to commit).

**Input:** `$ARGUMENTS` — Notion URL or page ID.

If missing, abort with:
```
Usage: /customer-dev:ship <notion-url-or-page-id>
```

---

## Flow overview

```
[IA]    Step 1 — Pre-flight (branch, status, gh auth)
[IA]    Step 2 — Invoke customer-dev:plan → writes plan-<TICKET>-<slug>.md
[IA]    Step 3 — Present plan summary
⏸️      CHECKPOINT 1 (human): approve plan / edit / cancel
[IA]    Step 4 — Invoke customer-dev:execute → branch + tests RED→GREEN + lint + typecheck + coverage + build
[IA]    Step 5 — Invoke customer-dev:review → React antipattern scan on the diff
[IA]    Step 6 — Present execution + review reports + diff preview
⏸️      CHECKPOINT 2 (human): approve commit / review diff / fix flagged issues / cancel
[IA]    Step 7 — Commit + push + gh pr create
[IA]    Step 8 — Invoke customer-dev:pr-ready → CI watch + Notion update
[IA]    Step 9 — Final report with PR URL
```

**Total human touchpoints**: 1 invoke + 2 approvals. Everything else is automated.

---

## Step 1 — Pre-flight validations

Run in parallel:

```bash
git branch --show-current
git status --porcelain
git fetch origin main --quiet
gh auth status
node --version
npm --version
```

Validate:
- ❌ Uncommitted changes → abort: "Commit or stash changes before running /customer-dev:ship."
- ❌ `gh` not authenticated → abort: "Run `gh auth login` first."
- ❌ Node < 20 → warn: "CI runs on Node 20. Local version is <X>."
- ⚠️ Already on a feature branch → ask: "Current branch is `<X>`. Continue here or switch to main? (continue/main/cancel)"

---

## Step 2 — Invoke `customer-dev:plan`

Use the `Skill` tool to invoke the planner:

```
Skill: customer-dev:plan
Args:  <notion-url from $ARGUMENTS>
```

The planner will:
1. Fetch the Notion task.
2. Load context (architecture, subscriptions, services, hooks, components, pages, testing, quality, react-mental-model, react-pitfalls, typescript-patterns, performance, rails-to-react-glossary).
3. Explore the codebase for analogues.
4. Validate ENV vars against `.env.example`.
5. Write `plan-<TICKET>-<slug>.md`.

**Capture**: the plan file path, the ticket ID, the subscription scope, and any `Open Questions` the planner flagged.

---

## Step 3 — Present plan summary

Show the user:

```
📋 Plan generated: plan-<TICKET>-<slug>.md

Summary:
  Ticket:        <TICKET>
  Title:         <title>
  Type:          <feature|bugfix|improvement|chore>
  Scope:         <subscription slug or "global">
  Files:         <N> to create, <N> to modify

ENV vars:        <list or "None">
i18n keys:       <list or "None">

Top 3 decisions made:
  1. <from planner>
  2. <from planner>
  3. <from planner>

Open questions (blockers):
  <list or "None">
```

If there are **Open Questions** the planner flagged as blockers, stop and present them. Do NOT proceed — ask the user to resolve them first.

---

## ⏸️ CHECKPOINT 1 — Plan approval (human)

Ask:

```
What next?
  1. Approve plan and start implementation
  2. Edit plan (I'll open it in your editor)
  3. Cancel

Your choice:
```

### Handling responses

- **1 (approve)** → continue to Step 4.
- **2 (edit)** → run `$EDITOR plan-<TICKET>-<slug>.md`. After the user saves, re-read the plan and re-present the summary. Then ask again.
- **3 (cancel)** → stop here. The plan file stays on disk for later use.

---

## Step 4 — Invoke `customer-dev:execute`

```
Skill: customer-dev:execute
Args:  <plan file path>
```

The executor will:
1. Read the plan.
2. Load the 13 context files.
3. Create the branch from `main`.
4. Write test files (RED) + implement until GREEN.
5. Run `npm run test:threshold` (per-file coverage gate).
6. Run `npm run lint` + `npm run format:check` + `npm run typecheck`.
7. Run `npm run build`.
8. **Does NOT commit** — ship handles that.

**Capture**: the execution report (files created/modified, tests status, coverage, lint/typecheck/build status, ENV vars applied).

If the executor stops (3 failure limit, plan blocker, coverage gate) → propagate the stop to the user. Do NOT proceed to review/commit.

---

## Step 5 — Invoke `customer-dev:review`

```
Skill: customer-dev:review
```

The reviewer scans the diff against `origin/main` for React/TS antipatterns and produces a report with three severity buckets: **BLOCKING**, **LIKELY BUGS**, **SUGGESTIONS**. It does NOT edit code.

**Capture**: counts per severity and the full report.

---

## Step 6 — Present execution report + review report + diff preview

Show:

```
✅ Execution completed

Files created (<N>):
  - <list>

Files modified (<N>):
  - <list>

Tests:       <N> new test files, all passing
Coverage:    per-file gate PASS (lines 75/funcs 65/stmts 75/branches 70)
Lint:        ✅
Format:      ✅
Typecheck:   ✅
Build:       ✅
ENV vars:    <list or "none">
i18n keys:   <list or "none">

🔍 React pre-review

  Blocking:     <N>
  Likely bugs:  <N>
  Suggestions:  <N>

<full review report text>
```

Then show the diff summary:

```bash
git diff origin/main...HEAD --stat
```

Offer to show the full diff:

```
Show full diff before committing? (y/n)
```

If yes → `git diff origin/main...HEAD` (user reviews).

**If review reports any BLOCKING or LIKELY BUGS**, surface them loudly. Default recommendation: resolve them before Checkpoint 2.

---

## ⏸️ CHECKPOINT 2 — Commit approval (human)

Propose the commit message (built from the plan):

```
Proposed commit:

  [<TYPE>] <Notion title> - <TICKET>

  Co-Authored-By: Claude <noreply@anthropic.com>

What next?
  1. Commit with this message, push, and open PR
  2. Edit commit message
  3. Fix review flags first (I'll stop; you iterate; re-run /customer-dev:ship or /customer-dev:execute)
  4. Review more (stop here, I'll commit manually)
  5. Cancel (rollback: discard implementation branch)

Your choice:
```

### Handling responses

- **1 (approve)** → continue to Step 7.
- **2 (edit)** → prompt for a new message (must still match `[<TYPE>] <desc> - <TICKET>` format). Validate before committing.
- **3 (fix flags)** → exit with the review report pinned. The branch stays; the user iterates.
- **4 (stop)** → present a summary of where things are. The user takes over manually. Exit.
- **5 (cancel)** → ask confirmation ("This will reset the branch and discard all changes. Sure? y/n"). If yes:
  ```bash
  git reset --hard origin/main
  git clean -fd
  git checkout main
  git branch -D <feature-branch>
  ```
  Then exit.

---

## Step 7 — Commit, push, open PR

Determine `TYPE` from the branch prefix:
- `feature/*` → `FEAT`
- `bugfix/*` or `hotfix/*` → `FIX`
- `improvement/*` → `REFACTOR`
- `chore/*` → `CHORE`

```bash
git add <files from execution report>
git commit -m "$(cat <<'EOF'
[<TYPE>] <title> - <TICKET>

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
git push -u origin <branch>

gh pr create --base main \
  --title "[<TYPE>] <title> - <TICKET>" \
  --body "$(cat <<'EOF'
## Summary

<from plan's Summary section>

## Ticket

[<TICKET>](<Notion URL>)

## Changes

<from execution report's files created/modified>

## Checklist

- [x] Tests written before implementation (RED → GREEN)
- [x] All Business Rules covered by tests
- [x] Coverage per-file gate PASS (75/65/75/70)
- [x] Lint + Format + Typecheck clean
- [x] Build passes
- [x] React pre-review: no BLOCKING (LIKELY BUGS resolved or justified)
- [x] No cross-subscription imports
- [x] i18n keys added for user-facing strings
- [x] ENV vars documented in .env.example (if any)

## Test plan

- [ ] Reviewer runs: \`npm test -- <paths>\`
- [ ] Reviewer runs: \`npm run quality:check && npm run build\`
- [ ] Reviewer exercises the UI flow manually
- [ ] Reviewer verifies ENV vars in deploy config (if listed in plan)
EOF
)"
```

**Capture**: the PR URL from `gh pr create` output.

---

## Step 8 — Invoke `customer-dev:pr-ready`

```
Skill: customer-dev:pr-ready
Args:  <notion-url or ticket ID>
```

The skill will:
1. Verify branch/PR state.
2. Squash commits if > 1 (force-with-lease).
3. Verify Notion task state.
4. Run `npm run quality:check && npm run build` locally (final gate).
5. Watch `gh pr checks --watch` for GitHub Actions green.
6. If all passes, ask to move the Notion task to `Code Review`.

---

## Step 9 — Final report

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🚀 Ship completed
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Ticket:       <TICKET> — <title>
Branch:       <branch name>
PR:           <PR URL>
Notion:       <previous status> → <new status>

Pipeline:
  Plan:         ✅ approved by human
  Execute:      ✅ <N> files created, tests passing, coverage clean
  Review:       ✅ 0 blocking, 0 likely bugs (or: N resolved)
  Commit:       ✅ <commit SHA>
  Push:         ✅ to origin/<branch>
  PR:           ✅ opened against main
  Quality gate: ✅ lint + format + typecheck + coverage + build
  GH Actions:   ✅ all 3 jobs green (quality / test / build)
  Notion:       ✅ moved to Code Review

Reviewers can now review:
  <PR URL>

Manual follow-ups (from executor / review report):
  <list or "None">
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Hard rules

- **Always** require 2 human checkpoints (plan + commit). Never auto-commit.
- **Never** proceed past a checkpoint if the user didn't explicitly approve.
- **Always** run `customer-dev:review` between execute and the commit checkpoint. The human sees the antipattern scan before approving the commit.
- **Never** ignore BLOCKING review flags silently — surface them and default to "fix before commit."
- **Never** force-push in ship (the PR is created fresh; squash happens later in pr-ready).
- **Always** include the Notion URL in the PR body for traceability.
- **If any invoked skill stops** (3 failure rule, open questions, blocker), propagate the stop with full context.
- **Checkpoint 2 option 5 (cancel)** requires explicit `yes` confirmation before destructive reset — don't assume.

---

## When NOT to use `/customer-dev:ship`

- You want only a plan (no implementation yet): use `/customer-dev:plan`.
- You have an existing plan file and want to execute only: use `/customer-dev:execute <plan.md>`.
- You want only the antipattern scan on your current branch: use `/customer-dev:review`.
- You already committed manually and just want to prep the PR for review: use `/customer-dev:pr-ready`.

`/customer-dev:ship` is for the **full new-task happy path**. For everything else, use the individual skills.
