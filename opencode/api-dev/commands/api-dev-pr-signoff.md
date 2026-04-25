---
description: Validate and sign off a PR in core-web-api: squash commits, run bin/ci, verify undercover diff coverage, and update the Notion task status to Code Review.
model: moonshotai/kimi-k2.6
---

# PR Signoff — core-web-api

Validates and completes the signoff workflow for a Pull Request before it goes to review. Squashes commits, runs `bin/ci`, verifies diff coverage with `undercover`, and updates the task status in Notion.

**Input:** optional `$ARGUMENTS` — Notion URL or page ID. If omitted, infers the ticket from the branch name.

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
- ✅ PR exists for the current branch.
- ❌ No PR → abort with: "Create the PR first: `gh pr create --base main`".
- ❌ Uncommitted changes → abort with: "Commit your changes first."

---

## Step 2 — Extract ticket ID

Parse the branch name for the ticket ID (regex: `/(KODIM-\d+|SC-\d+|[A-Z]+-\d+)/`).
If not found AND `$ARGUMENTS` is a Notion URL/ID → use it.
If neither → warn: "No ticket ID detected. Notion update will be skipped."

---

## Step 3 — Squash commits (if more than one)

If `COMMIT_COUNT > 1`:

1. Fetch task title from Notion (`mcp__notion__API-retrieve-a-page`).
2. Determine `TYPE` from branch prefix: `feature/*` → `FEAT`, `bugfix/*` or `hotfix/*` → `FIX`, `improvement/*` → `REFACTOR`, `chore/*` → `CHORE`.
3. Execute:
   ```bash
   git reset --soft $(git merge-base main HEAD)
   git commit -m "$(cat <<'EOF'
   [<TYPE>] <title> - <TICKET>

   Co-Authored-By: Claude <noreply@anthropic.com>
   EOF
   )"
   git push --force-with-lease
   ```

If `COMMIT_COUNT == 1`: verify message matches `[<TYPE>] <desc> - <TICKET>`. If not, amend + `--force-with-lease`.

**Always `--force-with-lease`, never `--force`.**

---

## Step 4 — Verify Notion task state

Call `mcp__notion__API-retrieve-a-page` and inspect the `Status` property.

- ✅ `In Progress` → OK, will transition after CI.
- ✅ `Code Review` → already there, no transition needed.
- ⚠️ Other → continue but inform.

---

## Step 5 — Run bin/ci

```bash
bin/ci
```

Runs: `setup` + `rubocop` + `brakeman` + `rspec` + signoff. Capture exit code.

---

## Step 6 — Diff coverage gate (blocking)

```bash
doppler run --config test -- COVERAGE=true bundle exec rspec
doppler run --config test -- bundle exec undercover --compare origin/main
```

If undercover reports uncovered lines → **do NOT transition to Code Review**. Report to the user:

```
❌ Undercover found N uncovered lines in the diff:
  - path/to/file.rb:42

Fix before requesting review:
  1. Add specs covering these lines, OR
  2. Wrap in `# :nocov:` with a comment justifying why.
```

---

## Step 7 — Handle bin/ci result

### ✅ If bin/ci PASSED

1. Confirm success.
2. Get PR URL: `gh pr view --json url -q .url`
3. If task is `In Progress`, ask: "Move it to Code Review? (y/n)"
4. If yes, call `mcp__notion__API-patch-page` to set `Status` → `Code Review`.

### ❌ If bin/ci FAILED

**RuboCop failed:**
```
Auto-fix with `bundle exec rubocop -A`? (y/n)
```
If yes: autofix, commit `[CHORE] Auto-fix rubocop - <TICKET>`, push, re-run `bin/ci`.

**Tests failed:** Show instructions to fix manually. Do NOT auto-fix.

**Brakeman failed:** Show instructions to fix manually. Do NOT auto-fix.

**Never** move to Code Review if `bin/ci` or `undercover` failed.

---

## Step 8 — Final summary

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 PR Signoff Summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Task:   <TICKET> — <title>
Notion: <previous status> → <new status>
PR:     <PR URL>

Squash: <✅ N → 1 | ⏭️ already 1>
bin/ci: ✅ / ❌
Undercover: ✅ clean / ❌ N uncovered

Next action: <PR ready for review | Fix and re-run>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Hard rules

- **Always** squash multi-commit branches before `bin/ci`.
- **Always** use `--force-with-lease`, never `--force`.
- **Never** move to Code Review if `bin/ci` OR `undercover` failed.
- **Never** auto-fix failing tests or Brakeman — humans own those.
- **Never** push to `main` directly.
