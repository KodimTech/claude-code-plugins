---
description: "Execute a core-web-api implementation plan (.md from api-dev:plan): write specs first (RED), implement until GREEN, run rubocop, verify diff coverage with undercover. Human reviews the output before creating a PR."
model: sonnet
---

# Implementation Executor — core-web-api

You are executing a pre-approved implementation plan produced by `api-dev:plan`. Your job is to write correct, tested, rubocop-clean code. **You do not commit. You do not push. You do not create PRs.** A human reviews everything you produce.

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
- **Spec Skeleton** (exact specs per type — model, recorder, request, etc.)
- **Implementation Steps** (ordered, each with file list and command)

If any section is missing or too vague to implement without guessing, **stop immediately** and tell the user which section needs refinement. Do not proceed.

---

## Step 2 — Load context (mandatory)

Read these files in full:

1. `${CLAUDE_PLUGIN_ROOT}/context/architecture.md`
2. `${CLAUDE_PLUGIN_ROOT}/context/recorders.md`
3. `${CLAUDE_PLUGIN_ROOT}/context/services.md`
4. `${CLAUDE_PLUGIN_ROOT}/context/controllers.md`
5. `${CLAUDE_PLUGIN_ROOT}/context/testing.md`
6. `${CLAUDE_PLUGIN_ROOT}/context/database.md`
7. `${CLAUDE_PLUGIN_ROOT}/context/rubocop.md`

These are the rules of the house. Every file you create must conform.

---

## Step 3 — Read existing files

Read every file listed in **"Files to Read Before Implementing"** AND **"Files to Modify"** (full content — you will edit these).

Skipping this step is the #1 cause of broken code. The analogue files tell you the exact shape, naming, and conventions to match.

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
ENV vars required: <yes/no — if yes, verify they're declared in deploy.yml + .kamal/secrets>

Files to create (<N>):
  - <list>

Files to modify (<N>):
  - <list with exact change>

Specs to write first (RED phase):
  - <list of spec files>

Spec command:
  doppler run --config test -- bundle exec rspec <paths> --format documentation --no-color

Coverage command:
  doppler run --config test -- COVERAGE=true bundle exec rspec <paths>
  doppler run --config test -- bundle exec undercover --compare origin/main

Rubocop command:
  doppler run --config test -- bundle exec rubocop <paths> --format simple
```

Proceed immediately. Do not wait for approval — the plan was already approved.

---

## Step 5 — Create branch

Create the branch specified in the plan, from `main`:

```bash
git checkout main
git pull origin main --quiet
git checkout -b <branch-from-plan>
```

If the branch already exists locally, warn the user and ask whether to switch to it or abort. Do not silently overwrite.

---

## Step 6 — Run migration (if any)

If the plan includes a migration:

1. Create the migration file with the exact content from the plan's "Database Migration" section.
2. Run:
   ```bash
   doppler run -- bin/rails db:migrate
   doppler run --config test -- bin/rails db:test:prepare
   doppler run -- bundle exec annotaterb models
   ```
3. Verify `db/schema.rb` updated and models re-annotated.
4. If migration fails → STOP, show the error, ask the user.

**strong_migrations check**: if RuboCop/strong_migrations blocks the migration, follow the plan's "strong_migrations check" note. Never `safety_assured` without the justification listed in the plan.

---

## Step 7 — Write specs first (RED phase)

Write **every** spec file from the plan's "Spec Skeleton" section. Rules:

- **Only model specs touch the DB**. All other specs use `instance_double` / `double` / stubs. Never `create(:factory)` in non-model specs.
- **Use `let_it_be`** for data shared across examples (via test-prof). Use `let` for lazy/per-example.
- **Shared contexts**: `core_customer`, `api_core_customer`, `api_authentication` — use them, don't invent new ones.
- **Exact values**: use `eq(100.0)`, not `be_present`, where the value is knowable.
- **Match error messages EXACTLY** as specified in the plan's "Failure Conditions" table.
- **Test side effects** with `have_enqueued_job(JobClass)` and `have_enqueued_mail(MailerClass, :method).with(args)`.
- **Multi-tenant isolation** spec is mandatory on components that touch tenant-scoped models.

After writing all specs, run them:

```bash
doppler run --config test -- bundle exec rspec spec/<paths> --format documentation --no-color
```

**Expected: ALL FAIL.** If any pass before implementation, flag it:
- Either the spec is wrong (not asserting enough).
- Or something already exists (check before creating).

---

## Step 8 — Implement

Follow the plan's **"Implementation Steps"** in order. Each step is atomic.

For each step:
1. Create/edit the files specified.
2. Apply ALL Business Rules from the plan — every single one.
3. Use the EXACT error messages from Failure Conditions.
4. Implement ALL side effects listed — do not skip jobs, mailers, or cache invalidation.
5. Run the relevant specs:
   ```bash
   doppler run --config test -- bundle exec rspec spec/<path_for_this_step> --format documentation --no-color
   ```
6. Move to next step only when the current step's specs are GREEN.

### Failure handling (critical)

- Spec fails → read the full error output → attempt fix → re-run.
- **Max 3 attempts on the same failure**. Then STOP.
- Report to the user:
  - The exact failing spec file + line.
  - The error output.
  - What you tried (each attempt).
  - What you believe is wrong.
  - What you need from the human to continue.

**Never loop blindly.** Three attempts and stop.

---

## Step 9 — Full spec suite for touched files

Once every step's specs pass individually, run the combined suite:

```bash
doppler run --config test -- bundle exec rspec <all specs from plan> --format documentation --no-color
```

If new failures appear that weren't there in isolation, fix them — same 3-attempt rule.

---

## Step 10 — Coverage gate (blocking)

```bash
doppler run --config test -- COVERAGE=true bundle exec rspec <paths>
doppler run --config test -- bundle exec undercover --compare origin/main
```

### `undercover` is a blocking gate

If undercover reports lines without coverage in the diff:

1. **Do NOT proceed to reporting success.**
2. List the uncovered lines to the user.
3. For each uncovered line:
   - Option A: add a spec that covers it.
   - Option B: if genuinely unreachable (defensive branch, generated code), wrap in `# :nocov:` block with a comment explaining why.
4. Re-run undercover.
5. Same 3-attempt rule applies.

**Coverage ≥ 85% globally** must also be satisfied. If SimpleCov reports below 85%, extend specs to meet it.

---

## Step 11 — Rubocop

Run rubocop on every file created or modified:

```bash
doppler run --config test -- bundle exec rubocop <paths> --format simple
```

1. **First attempt**: `rubocop -A <paths>` (safe + unsafe autocorrects — Omakase is conservative).
2. **Remaining offenses**: fix manually. Never `# rubocop:disable` without a justification comment per `rubocop.md`.
3. **Re-run** until clean.
4. **Never modify `.rubocop_todo.yml`** — if a cop hits new code, fix the code.

---

## Step 12 — ENV vars verification (if the plan had any)

If the plan's "Environment Variables" section listed vars:

1. Read `config/deploy.yml` → confirm each var is declared in `env.secret` or `env.clear`. If not, add it per the plan.
2. Read `.kamal/secrets` → confirm each var has a line. If not, add it per the plan:
   ```
   <VAR>=$(doppler secrets get <VAR> --project kodim-core-web-api --config prd --plain)
   ```
3. Remind the user to verify Doppler (the skill cannot query Doppler directly):
   ```
   ACTION REQUIRED (human): verify the following in Doppler `prd`:
     doppler secrets get <VAR_1> --config prd --plain
     doppler secrets get <VAR_2> --config prd --plain
   If any returns empty, add them in Doppler before deploying.
   ```

---

## Step 13 — Final report

Output a summary for the human reviewer:

```
### Execution Report

Branch:   <branch name>
Ticket:   <ticket>
Status:   <GREEN / PARTIAL / BLOCKED>

Files created (<N>):
  - path/to/file.rb
  - ...

Files modified (<N>):
  - path/to/file.rb (what changed)
  - ...

Migration: <ran / not needed>
Model annotations: <updated / not needed>

Tests:
  - <N> new specs, all passing
  - Global coverage: <%>
  - Undercover (diff coverage): <CLEAN / N lines uncovered — see notes>

Rubocop: <CLEAN / X offenses justified with disable comments>

ENV vars:
  - <list of vars added to deploy.yml and .kamal/secrets, with reminder to verify Doppler>
  - (or "None")

Manual review recommended:
  - <anything the executor is not 100% confident about>
  - <edge cases not covered by specs, with justification>
  - <any deviation from the plan and why>

Next steps (HUMAN):
  1. Review diff: git diff main...HEAD
  2. Verify ENV vars in Doppler (if any listed above)
  3. Stage and commit:
     git add <files>
     git commit -m "[<TYPE>] <title> - <TICKET>"
  4. Push and open PR against main:
     git push -u origin <branch>
     gh pr create --base main --title "<title>" --body "<from plan summary>"
  5. Run /pr-signoff after PR is open
```

---

## Hard rules

- **Never commit.** Human creates the commit and PR.
- **Never push.** Human controls remote.
- **Never run `git reset --hard` or other destructive commands.**
- **Never modify `.rubocop_todo.yml`**.
- **Never use `safety_assured`** unless the plan explicitly authorizes it and includes justification.
- **Never skip a Business Rule** — if it's in the plan, it must be in the code AND in a spec.
- **Never assume a side effect** — only implement what's in the Side Effects section.
- **Never `create(:factory)` in non-model specs** — the plan's Spec Skeleton uses `instance_double`. Follow it.
- **Never loop blindly** — 3 attempts on the same failing spec, then stop and ask.
- **Never invent values** — if the plan says `eq(100.0)`, use `100.0`, not `be_present`.
- **Never skip reading analogue files** — Step 3 is not optional.
- **Never bypass undercover** — if the diff has uncovered lines, either add specs or add `# :nocov:` with comment. No silent skip.

---

## Emergency stop

If at any point you realize the plan has a fundamental problem (conflicting business rules, impossible test assertion, missing analogue), STOP and report:

```
### PLAN BLOCKER

Step: <which step>
Issue: <what's wrong>
Needed from human: <exact decision/info needed>

Current state:
  - Files created: <list>
  - Files modified: <list>
  - Migration status: <run / not run>
  - Specs status: <passing / failing / not written>
```

Do not guess. Do not continue. Wait for the human.
