---
description: "Validate and sign off a PR in core-web-api: squash commits, run bin/ci (rspec + rubocop + brakeman + signoff), verify undercover diff coverage, and update the Notion task status."
model: sonnet
---

# PR Signoff — core-web-api

Validates and completes the signoff workflow for a Pull Request before it goes to review. Squashes commits, runs `bin/ci`, verifies diff coverage with `undercover`, and updates the task status in Notion.

**Input:** optional `$ARGUMENTS` — Notion URL or page ID. If omitted, the skill tries to infer the ticket from the branch name.

---

## Step 1 — Pre-flight validations

Run in parallel:

```bash
git branch --show-current
gh pr view --json url,title,state
git status --porcelain
git rev-list --count main..HEAD
```

Validate:
- ✅ Branch matches `<prefix>/<slug>-<TICKET>` where prefix ∈ `feature/`, `bugfix/`, `improvement/`, `hotfix/`, `chore/`.
- ❌ If on `main` → abort.
- ✅ PR exists for the current branch (`gh pr view`).
- ❌ No PR → abort with: "Create the PR first: `gh pr create --base main`".
- ❌ Uncommitted changes → abort with: "Commit your changes first."
- Capture commit count for step 3.

---

## Step 2 — Extract ticket ID

Parse the branch name for the ticket ID:
- Regex: `/(KODIM-\d+|SC-\d+|[A-Z]+-\d+)/`
- If found → use it.
- If not found AND `$ARGUMENTS` is a Notion URL/ID → use it.
- If neither → warn the user: "No ticket ID detected. Notion update will be skipped."

---

## Step 3 — Squash commits (if more than one)

```bash
COMMIT_COUNT=$(git rev-list --count main..HEAD)
```

If `COMMIT_COUNT > 1`:

1. Fetch task details from Notion (if ticket ID detected):
   - `mcp__notion__API-retrieve-a-page` with the `page_id`.
   - Extract the title for the commit message.

2. Determine `TYPE` from the branch prefix:
   - `feature/*` → `FEAT`
   - `bugfix/*` or `hotfix/*` → `FIX`
   - `improvement/*` → `REFACTOR`
   - `chore/*` → `CHORE`

3. Build the squash commit message:
   ```
   [<TYPE>] <Notion title or branch slug> - <TICKET>

   Co-Authored-By: Claude <noreply@anthropic.com>
   ```

4. Execute the squash:
   ```bash
   git reset --soft $(git merge-base main HEAD)
   git commit -m "$(cat <<'EOF'
   [<TYPE>] <title> - <TICKET>

   Co-Authored-By: Claude <noreply@anthropic.com>
   EOF
   )"
   git push --force-with-lease
   ```

   **Use `--force-with-lease`, never `--force`** — protects against overwriting remote changes someone else pushed.

If `COMMIT_COUNT == 1`:
- Verify the message matches `[<TYPE>] <desc> - <TICKET>` format.
- If not, run `git commit --amend` with a corrected message (+ `git push --force-with-lease`).
- If it matches, proceed without changes.

---

## Step 4 — Verify Notion task state (if ticket detected)

Call `mcp__notion__API-retrieve-a-page` with the ticket's `page_id` and inspect the `Status` property.

Valid states for PR phase:
- ✅ `In Progress` → OK, will transition to Code Review after successful CI.
- ✅ `Code Review` → already there, no transition needed.
- ⚠️ `To Do` / `Backlog` → warn: "Task isn't in progress — consider moving it to In Progress first."
- ⚠️ Other → continue but inform.

---

## Step 5 — Run bin/ci

```bash
bin/ci
```

This executes: `setup` + `rubocop` + `brakeman` + `rspec` + auto-signoff (via `gh signoff`).

**Capture exit code.**

---

## Step 6 — Diff coverage gate (blocking)

After `bin/ci` passes, run the additional coverage check:

```bash
doppler run --config test -- COVERAGE=true bundle exec rspec
doppler run --config test -- bundle exec undercover --compare origin/main
```

If `undercover` reports uncovered lines in the diff → **do NOT transition the task to Code Review**. Report to the user:

```
❌ Undercover found N uncovered lines in the diff:
  - path/to/file.rb:42
  - path/to/other.rb:17

Fix before requesting review:
  1. Add specs covering these lines, OR
  2. Wrap in `# :nocov:` with a comment justifying why (rare — prefer specs).
```

---

## Step 7 — Handle bin/ci result

### ✅ If bin/ci PASSED (exit 0)

1. Confirm success to the user:
   ```
   ✅ bin/ci passed
     - Tests: ✅
     - RuboCop: ✅
     - Brakeman: ✅
     - Signoff: ✅
     - Undercover (diff coverage): ✅ clean
   ```

2. Get the PR URL:
   ```bash
   gh pr view --json url -q .url
   ```

3. If task is in `In Progress`, ask:
   ```
   Task is in "In Progress". Move it to "Code Review"? (y/n)
   ```

4. If yes, update Notion:
   - Call `mcp__notion__API-patch-page` with the ticket's `page_id`.
   - Set the `Status` property to `Code Review` (the exact option name must match the Notion database schema — verify via `mcp__notion__API-retrieve-a-database` on first run).

### ❌ If bin/ci FAILED (exit != 0)

1. Analyze output to identify the failing stage:

   **RuboCop failed:**
   ```
   ❌ RuboCop found style issues.
   Auto-fix with `bundle exec rubocop -A`? (y/n)
   ```
   If yes: run autofix, stage, commit `[CHORE] Auto-fix rubocop - <TICKET>`, push, re-run `bin/ci`.

   **Tests failed:**
   ```
   ❌ Tests failed.

   Fix manually:
     1. Review output above
     2. doppler run --config test -- bundle exec rspec <failing_spec>
     3. Fix code
     4. Commit + push
     5. Re-run /api-dev:pr-signoff
   ```
   Do NOT auto-fix tests — that's the human's job.

   **Brakeman failed:**
   ```
   ❌ Brakeman found security warnings.

   Fix manually:
     1. Review output above
     2. Fix unsafe code
     3. Commit + push
     4. Re-run /api-dev:pr-signoff
   ```

2. **NEVER** move the task to Code Review if `bin/ci` or `undercover` failed.

---

## Step 8 — Final summary

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 PR Signoff Summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Task:   <TICKET> — <title>
Notion: <previous status> → <new status>
PR:     <PR URL>

Preparation:
  Squash: <✅ N commits → 1 commit | ⏭️ already 1 commit>

Validations:
  bin/ci (rspec + rubocop + brakeman + signoff): ✅ / ❌
  Undercover diff coverage: ✅ clean / ❌ N uncovered

Next action:
  <e.g., "PR ready for review, pinged reviewers in Slack" | "Fix failing tests and re-run">
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Hard rules

- **Always** run pre-flight validations before `bin/ci` — fail fast.
- **Always** squash multi-commit branches before running `bin/ci` — one commit per PR.
- **Always** use `--force-with-lease`, never `--force`.
- **Never** move the task to Code Review if `bin/ci` OR `undercover` failed.
- **Never** auto-fix failing tests or Brakeman — humans own those.
- **Never** bypass undercover — diff coverage is part of the gate.
- **Never** push to `main` directly.
- **Never** skip reading the Notion task state — if a ticket ID is detected, check Notion.
