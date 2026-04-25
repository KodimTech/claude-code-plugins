---
description: Execute a core-web-api implementation plan (.md from /api-dev-plan): write specs first (RED), implement until GREEN, run rubocop, verify diff coverage with undercover. Human reviews the output before committing.
model: moonshotai/kimi-k2.6
---

# Implementation Executor — core-web-api

You are executing a pre-approved implementation plan produced by `/api-dev-plan`. Your job is to write correct, tested, rubocop-clean code. **You do not commit. You do not push. You do not create PRs.** A human reviews everything you produce.

**Input:** `$ARGUMENTS` — path to the plan `.md` file (e.g., `plan-KODIM-456-add-item-variants.md`).

---

## Step 1 — Read the plan

Read the file at `$ARGUMENTS` in full. Extract and hold in memory:

- **Ticket ID** and **Branch name**
- **Module** and **Audience**
- **Files to Read Before Implementing**
- **Files to Create**
- **Files to Modify**
- **Database Migration** (full ruby block, post-migration commands)
- **Environment Variables** (if any — if missing in deploy.yml / .kamal/secrets, STOP and ask)
- **Business Rules** (all of them)
- **Failure Conditions** (condition → exact error message)
- **Side Effects** (jobs, mailers, cache invalidation)
- **Spec Skeleton** (exact specs per type)
- **Implementation Steps** (ordered, each with file list and command)

If any section is missing or too vague to implement without guessing, **stop immediately** and tell the user which section needs refinement. Do not proceed.

---

## Step 2 — Load context (mandatory)

Read these files in full:

1. `.opencode/context/architecture.md`
2. `.opencode/context/recorders.md`
3. `.opencode/context/services.md`
4. `.opencode/context/controllers.md`
5. `.opencode/context/testing.md`
6. `.opencode/context/database.md`
7. `.opencode/context/rubocop.md`

These are the rules of the house. Every file you create must conform.

---

## Step 3 — Read existing files

Read every file listed in **"Files to Read Before Implementing"** AND **"Files to Modify"** (full content — you will edit these).

Skipping this step is the #1 cause of broken code.

---

## Step 4 — Show execution plan

Before touching any file, output:

```
### Execution Plan

Branch:   <from plan>
Ticket:   <from plan>
Module:   <from plan>
Audience: <from plan>

Migration needed: <yes/no>
ENV vars required: <yes/no>

Files to create (<N>):
  - <list>

Files to modify (<N>):
  - <list with exact change>

Specs to write first (RED phase):
  - <list of spec files>
```

Proceed immediately. Do not wait for approval — the plan was already approved.

---

## Step 5 — Create branch

```bash
git checkout main
git pull origin main --quiet
git checkout -b <branch-from-plan>
```

If the branch already exists locally, warn the user and ask whether to switch to it or abort.

---

## Step 6 — Run migration (if any)

1. Create the migration file with the exact content from the plan's "Database Migration" section.
2. Run:
   ```bash
   doppler run -- bin/rails db:migrate
   doppler run --config test -- bin/rails db:test:prepare
   doppler run -- bundle exec annotaterb models
   ```
3. Verify `db/schema.rb` updated and models re-annotated.
4. If migration fails → STOP, show the error, ask the user.

---

## Step 7 — Write specs first (RED phase)

Write **every** spec file from the plan's "Spec Skeleton" section. Rules:

- **Only model specs touch the DB**. All other specs use `instance_double`. Never `create(:factory)` in non-model specs.
- **Shared contexts**: `core_customer`, `api_core_customer`, `api_authentication` — use them.
- **Exact values**: use `eq(100.0)`, not `be_present`.
- **Match error messages EXACTLY** as specified in "Failure Conditions".
- **Test side effects** with `have_enqueued_job` and `have_enqueued_mail`.
- **Multi-tenant isolation** spec is mandatory on components touching tenant-scoped models.

After writing all specs, run them:

```bash
doppler run --config test -- bundle exec rspec spec/<paths> --format documentation --no-color
```

**Expected: ALL FAIL.** If any pass before implementation, flag it.

---

## Step 8 — Implement

Follow the plan's **"Implementation Steps"** in order. Each step is atomic.

For each step:
1. Create/edit the files specified.
2. Apply ALL Business Rules from the plan.
3. Use the EXACT error messages from Failure Conditions.
4. Implement ALL side effects listed.
5. Run the relevant specs:
   ```bash
   doppler run --config test -- bundle exec rspec spec/<path_for_this_step> --format documentation --no-color
   ```
6. Move to next step only when the current step's specs are GREEN.

### Failure handling

- Spec fails → read full error → attempt fix → re-run.
- **Max 3 attempts on the same failure**. Then STOP.
- Report: exact failing spec + line, error output, what you tried, what you need from the human.

**Never loop blindly.** Three attempts and stop.

---

## Step 9 — Full spec suite for touched files

```bash
doppler run --config test -- bundle exec rspec <all specs from plan> --format documentation --no-color
```

---

## Step 10 — Coverage gate (blocking)

```bash
doppler run --config test -- COVERAGE=true bundle exec rspec <paths>
doppler run --config test -- bundle exec undercover --compare origin/main
```

If undercover reports uncovered lines in the diff:
1. **Do NOT proceed.**
2. List uncovered lines.
3. Add spec covering each, OR wrap in `# :nocov:` with a justification comment.
4. Re-run undercover. Same 3-attempt rule.

**Coverage ≥ 85% globally** must also be satisfied.

---

## Step 11 — Rubocop

```bash
doppler run --config test -- bundle exec rubocop <paths> --format simple
```

1. First: `rubocop -A <paths>` (safe + unsafe autocorrects).
2. Remaining offenses: fix manually.
3. **Never modify `.rubocop_todo.yml`**.

---

## Step 12 — ENV vars verification (if the plan had any)

1. Read `config/deploy.yml` → confirm each var is declared. If not, add it per the plan.
2. Read `.kamal/secrets` → confirm each var has a line. If not, add it per the plan.
3. Remind the user:
   ```
   ACTION REQUIRED (human): verify in Doppler `prd`:
     doppler secrets get <VAR> --config prd --plain
   ```

---

## Step 13 — Final report

```
### Execution Report

Branch:   <branch name>
Ticket:   <ticket>
Status:   <GREEN / PARTIAL / BLOCKED>

Files created (<N>):
  - path/to/file.rb

Files modified (<N>):
  - path/to/file.rb (what changed)

Migration: <ran / not needed>
Tests:     <N> new specs, all passing
Coverage:  <%> global / diff: clean
Rubocop:   <CLEAN / X offenses justified>
ENV vars:  <list or "None">

Next steps (HUMAN):
  1. Review diff: git diff main...HEAD
  2. Verify ENV vars in Doppler (if any)
  3. Stage and commit:
     git add <files>
     git commit -m "[<TYPE>] <title> - <TICKET>"
  4. Push and open PR:
     git push -u origin <branch>
     gh pr create --base main --title "<title>"
  5. Run /api-dev-pr-signoff
```

---

## Hard rules

- **Never commit.** Human creates the commit and PR.
- **Never push.**
- **Never modify `.rubocop_todo.yml`**.
- **Never use `safety_assured`** unless the plan explicitly authorizes it.
- **Never skip a Business Rule**.
- **Never `create(:factory)` in non-model specs**.
- **Never loop blindly** — 3 attempts then stop and ask.
- **Never bypass undercover**.
