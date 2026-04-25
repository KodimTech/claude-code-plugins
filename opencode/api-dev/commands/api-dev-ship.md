---
description: End-to-end orchestrator for a Notion task in core-web-api. Runs plan → human checkpoint → execute → human checkpoint → commit/push/PR → pr-signoff. Two explicit human checkpoints.
model: moonshotai/kimi-k2.6
---

# Ship — Full-flow orchestrator

Chains the full `api-dev` workflow in sequence with **two explicit human checkpoints**. Goal: ship a Notion task end-to-end with minimal friction, while keeping the human in control of the two decisions that matter (is the plan good, and is the code ready to commit).

**Input:** `$ARGUMENTS` — Notion URL or page ID.

If missing, abort with:
```
Usage: /api-dev-ship <notion-url-or-page-id>
```

---

## Flow overview

```
[AI]    Step 1 — Pre-flight (branch, status, doppler, gh auth)
[AI]    Step 2 — Run planning workflow → writes plan-<TICKET>-<slug>.md
[AI]    Step 3 — Present plan summary
⏸️      CHECKPOINT 1 (human): approve plan / edit / cancel
[AI]    Step 4 — Run execution workflow → branch + migration + specs + code + rubocop + undercover
[AI]    Step 5 — Present execution report + diff preview
⏸️      CHECKPOINT 2 (human): approve commit / review diff / cancel
[AI]    Step 6 — Commit + push + gh pr create
[AI]    Step 7 — Run pr-signoff workflow → bin/ci + Notion update
[AI]    Step 8 — Final report with PR URL
```

**Total human touchpoints**: 1 invoke + 2 approvals.

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
- ❌ Uncommitted changes → abort: "Commit or stash changes before running /api-dev-ship."
- ❌ `doppler` not installed → abort with install instructions.
- ❌ `gh` not authenticated → abort: "Run `gh auth login` first."
- ⚠️ Already on a feature branch → ask: "Current branch is `<X>`. Continue here or switch to main? (continue/main/cancel)"

---

## Step 2 — Run planning workflow

Execute the full planning workflow using the Notion task at `$ARGUMENTS`.

Follow exactly the steps from `/api-dev-plan`:
1. Validate git environment.
2. Fetch the Notion task (title, description, acceptance criteria, ticket ID, env vars).
3. Load context: read `.opencode/context/architecture.md`, `recorders.md`, `services.md`, `controllers.md`, `testing.md`, `database.md`, `rubocop.md` in full.
4. Explore the codebase for analogous files.
5. Show analysis block.
6. Validate ENV vars statically (reads `config/deploy.yml` and `.kamal/secrets`).
7. Write `plan-<TICKET>-<slug>.md`.

**Capture**: plan file path, ticket ID, classification, open questions.

---

## Step 3 — Present plan summary

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
  1. <from planning>
  2. <from planning>
  3. <from planning>

Open questions (blockers):
  <list or "None">
```

If there are **open questions flagged as blockers**, stop and present them. Do NOT proceed — ask the user to resolve them first.

---

## ⏸️ CHECKPOINT 1 — Plan approval (human)

Ask:

```
What next?
  1. Approve plan and start implementation
  2. Edit plan (open plan file and make changes, then reply when ready)
  3. Cancel

Your choice:
```

- **1 (approve)** → continue to Step 4.
- **2 (edit)** → wait for the user to edit and reply, then re-read the plan and re-present the summary. Ask again.
- **3 (cancel)** → stop. The plan file stays on disk for later use with `/api-dev-execute`.

---

## Step 4 — Run execution workflow

Execute the full implementation workflow using the plan file from Step 2.

Follow exactly the steps from `/api-dev-execute`:
1. Re-read the plan in full.
2. Load context (same 7 files as Step 2).
3. Read all analogue files from "Files to Read Before Implementing".
4. Show execution plan.
5. Create branch from `main`.
6. Run migration (if any) + annotaterb.
7. Write all specs (RED phase) and verify ALL FAIL.
8. Implement step-by-step, running specs after each step (GREEN phase). Max 3 attempts per failure.
9. Run full combined spec suite.
10. Run coverage gate: `COVERAGE=true rspec` + `undercover --compare origin/main` (blocking).
11. Run rubocop: `rubocop -A` then fix remaining manually.
12. Verify ENV vars in deploy.yml and .kamal/secrets (if any).

**Does NOT commit** — ship handles that.

If the executor stops (3 failure limit, plan blocker, undercover blocked) → propagate the stop to the user. Do NOT proceed to commit.

**Capture**: execution report (files created/modified, specs status, coverage, rubocop, ENV vars).

---

## Step 5 — Present execution report + diff preview

```
✅ Execution completed

Files created (<N>):
  - <list>

Files modified (<N>):
  - <list>

Migration:   <ran / not needed>
Specs:       <N> passing, 0 failing
Coverage:    <%> global / diff: clean
Rubocop:     clean
ENV vars:    <list or "none">

Manual review recommended:
  - <from executor>
```

Then show:
```bash
git diff main...HEAD --stat
```

Ask: "Show full diff before committing? (y/n)"

---

## ⏸️ CHECKPOINT 2 — Commit approval (human)

Propose the commit message:

```
Proposed commit:

  [<TYPE>] <Notion title> - <TICKET>

  Co-Authored-By: Claude <noreply@anthropic.com>

What next?
  1. Commit with this message, push, and open PR
  2. Edit commit message
  3. Stop here (I'll commit manually)
  4. Cancel (discard implementation branch)

Your choice:
```

- **1 (approve)** → continue to Step 6.
- **2 (edit)** → prompt for new message (must match `[<TYPE>] <desc> - <TICKET>`). Validate format before committing.
- **3 (stop)** → summarize where things are. User takes over. Exit.
- **4 (cancel)** → ask confirmation: "This will reset the branch and discard all changes. Sure? (y/n)". If yes:
  ```bash
  git reset --hard origin/main
  git clean -fd
  git checkout main
  git branch -D <feature-branch>
  ```

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

- [ ] Reviewer runs: `doppler run --config test -- bundle exec rspec <paths>`
- [ ] Reviewer verifies ENV vars in Doppler prd (if listed in plan)
EOF
)"
```

**Capture**: PR URL from `gh pr create` output.

---

## Step 7 — Run pr-signoff workflow

Execute the full pr-signoff workflow:
1. Verify branch + PR state.
2. Squash commits if needed (likely just 1 — skip).
3. Verify Notion task state.
4. Run `bin/ci`.
5. Run `undercover --compare origin/main`.
6. If all passes, ask to move Notion task to `Code Review`.

---

## Step 8 — Final report

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🚀 Ship completed
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Ticket:       <TICKET> — <title>
Branch:       <branch name>
PR:           <PR URL>
Notion:       <previous status> → Code Review

Pipeline:
  Plan:         ✅ approved by human
  Execute:      ✅ <N> files, specs passing, coverage clean
  Commit:       ✅ <commit SHA>
  Push:         ✅ to origin/<branch>
  PR:           ✅ opened against main
  bin/ci:       ✅
  Undercover:   ✅ diff covered
  Notion:       ✅ moved to Code Review

Reviewers can now review:
  <PR URL>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Hard rules

- **Always** require 2 human checkpoints (plan + commit). Never auto-commit.
- **Never** proceed past a checkpoint without explicit approval.
- **Never** bypass `undercover` — if execution reports uncovered lines, ship stops.
- **Never** force-push. The PR is created fresh.
- **Always** include the Notion URL in the PR body.
- **If any workflow step stops** (3 failure rule, open questions, blocker), propagate the stop with full context.
- **Checkpoint 2 option 4 (cancel)** requires explicit `yes` before destructive reset.
