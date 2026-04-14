---
description: "End-to-end orchestrator for a Notion task in core-web-api: runs api-dev:plan → human checkpoint → api-dev:execute → human checkpoint → commit/push/PR → api-dev:pr-signoff. Two explicit checkpoints (plan approval, commit approval); everything else is automated."
model: opus
---

# Ship — Full-flow orchestrator

Chains the 3 `api-dev` skills in sequence with **two explicit human checkpoints**. The goal: ship a Notion task end-to-end with minimal friction, while keeping the human in control of the two decisions that matter (is the plan good, and is the code ready to commit).

**Input:** `$ARGUMENTS` — Notion URL or page ID.

If missing, abort with:
```
Usage: /api-dev:ship <notion-url-or-page-id>
```

---

## Flow overview

```
[IA]    Step 1 — Pre-flight (branch, status, doppler auth)
[IA]    Step 2 — Invoke api-dev:plan → writes plan-<TICKET>-<slug>.md
[IA]    Step 3 — Present plan summary
⏸️      CHECKPOINT 1 (human): approve plan / edit / cancel
[IA]    Step 4 — Invoke api-dev:execute → branch + migration + specs + code + rubocop + undercover
[IA]    Step 5 — Present execution report + diff preview
⏸️      CHECKPOINT 2 (human): approve commit / review diff / cancel
[IA]    Step 6 — Commit + push + gh pr create
[IA]    Step 7 — Invoke api-dev:pr-signoff → CI + Notion update
[IA]    Step 8 — Final report with PR URL
```

**Total human touchpoints**: 1 invoke + 2 approvals. Everything else is automated.

---

## Step 1 — Pre-flight validations

Run in parallel:

```bash
git branch --show-current
git status --porcelain
git fetch origin main --quiet
doppler --version
gh auth status
```

Validate:
- ❌ Uncommitted changes → abort: "Commit or stash changes before running /api-dev:ship."
- ❌ `doppler` not installed → abort with install instructions.
- ❌ `gh` not authenticated → abort: "Run `gh auth login` first."
- ⚠️ Already on a feature branch → ask: "Current branch is `<X>`. Continue here or switch to main? (continue/main/cancel)"

---

## Step 2 — Invoke `api-dev:plan`

Use the `Skill` tool to invoke the planner:

```
Skill: api-dev:plan
Args:  <notion-url from $ARGUMENTS>
```

The planner will:
1. Fetch the Notion task.
2. Load context, explore codebase.
3. Validate ENV vars statically.
4. Write `plan-<TICKET>-<slug>.md`.

**Capture**: the plan file path, the analyzed classification, the ticket ID, and any `Open Questions` the planner flagged.

---

## Step 3 — Present plan summary

Show the user:

```
📋 Plan generated: plan-<TICKET>-<slug>.md

Summary:
  Ticket:     <TICKET>
  Title:      <title>
  Type:       <feature|bugfix|...>
  Module:     <module>
  Audience:   <audience>
  Components: <list>

Migration:   <yes/no>
ENV vars:    <list or "None">

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

## Step 4 — Invoke `api-dev:execute`

```
Skill: api-dev:execute
Args:  <plan file path>
```

The executor will:
1. Read the plan.
2. Load context.
3. Create the branch from `main`.
4. Run migration (if any) + `annotaterb`.
5. Write specs (RED) + implement (GREEN).
6. Run `rubocop -A` + manual fixes.
7. Run `undercover --compare origin/main` (blocking gate).
8. **Does NOT commit** — the ship skill handles that.

**Capture**: the execution report (files created/modified, specs status, coverage, rubocop status, ENV vars applied).

If the executor stops (3 failure limit, plan blocker, undercover blocked) → propagate the stop to the user. Do NOT proceed to commit.

---

## Step 5 — Present execution report + diff preview

Show:

```
✅ Execution completed

Files created (<N>):
  - <list>

Files modified (<N>):
  - <list>

Migration:   <ran / not needed>
Specs:       <N> passing, 0 failing
Coverage:    <% global> / <diff: clean>
Rubocop:     clean
ENV vars:    <list or "none">

Manual review recommended:
  - <from executor>
```

Then show the diff summary:

```bash
git diff main...HEAD --stat
```

Offer to show the full diff:

```
Show full diff before committing? (y/n)
```

If yes → `git diff main...HEAD` (user reviews).

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
  3. Review more (stop here, I'll commit manually)
  4. Cancel (rollback: discard implementation branch)

Your choice:
```

### Handling responses

- **1 (approve)** → continue to Step 6.
- **2 (edit)** → prompt for a new message (must still match `[<TYPE>] <desc> - <TICKET>` format). Validate before committing.
- **3 (stop)** → present a summary of where things are. The user takes over manually. Exit.
- **4 (cancel)** → ask confirmation ("This will reset the branch and discard all changes. Sure? y/n"). If yes:
  ```bash
  git reset --hard origin/main
  git clean -fd
  git checkout main
  git branch -D <feature-branch>
  ```
  Then exit.

---

## Step 6 — Commit, push, open PR

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

- [x] Specs written before implementation (RED → GREEN)
- [x] All Business Rules covered by specs
- [x] Multi-tenant isolation verified (where applicable)
- [x] Undercover diff coverage: clean
- [x] Rubocop: clean
- [x] ENV vars synced (if any)

## Test plan

- [ ] Reviewer runs: \`doppler run --config test -- bundle exec rspec <paths>\`
- [ ] Reviewer verifies ENV vars in Doppler prd (if listed in plan)
EOF
)"
```

**Capture**: the PR URL from `gh pr create` output.

---

## Step 7 — Invoke `api-dev:pr-signoff`

```
Skill: api-dev:pr-signoff
Args:  <notion-url or ticket ID>
```

The signoff will:
1. Verify branch/PR state.
2. Squash commits (in this case, likely just 1 — no squash needed).
3. Verify Notion task state.
4. Run `bin/ci`.
5. Re-run `undercover` (redundant safety check).
6. If all passes, ask to move the Notion task to `Code Review`.

---

## Step 8 — Final report

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
  Execute:      ✅ <N> files created, specs passing, coverage clean
  Commit:       ✅ <commit SHA>
  Push:         ✅ to origin/<branch>
  PR:           ✅ opened against main
  bin/ci:       ✅
  Undercover:   ✅ diff covered
  Notion:       ✅ moved to Code Review

Reviewers can now review:
  <PR URL>

Manual follow-ups (from executor report):
  <list or "None">
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Hard rules

- **Always** require 2 human checkpoints (plan + commit). Never auto-commit.
- **Never** proceed past a checkpoint if the user didn't explicitly approve.
- **Never** bypass `undercover` — if execute reports uncovered lines, ship stops.
- **Never** force-push in ship. The PR is created fresh.
- **Always** include the Notion URL in the PR body for traceability.
- **If any invoked skill stops** (3 failure rule, open questions, blocker), propagate the stop with full context to the user.
- **Checkpoint 2 option 4 (cancel)** requires explicit `yes` confirmation before destructive reset — don't assume.

---

## When NOT to use `/api-dev:ship`

- You want only a plan (no implementation yet): use `/api-dev:plan`.
- You have an existing plan file and want to execute only: use `/api-dev:execute <plan.md>`.
- You already committed manually and just want signoff: use `/api-dev:pr-signoff`.

`/api-dev:ship` is for the **full new-task happy path**. For everything else, use the individual skills.
