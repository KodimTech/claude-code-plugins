---
description: "Validate and sign off a PR in core-web-customer: squash commits, run the local quality gate (lint + format + typecheck + test:threshold + build), watch GitHub Actions CI (quality / test / build), and update the Notion task status."
model: sonnet
---

# PR Signoff — core-web-customer

Validates and completes the signoff workflow for a Pull Request before it goes to review. Squashes commits, runs the local quality gate, watches the 3-job GitHub Actions pipeline, and updates the task status in Notion.

**Input:** optional `$ARGUMENTS` — Notion URL or page ID. If omitted, the skill tries to infer the ticket from the branch name.

---

## Step 1 — Pre-flight validations

Run in parallel:

```bash
git branch --show-current
gh pr view --json url,title,state,number
git status --porcelain
git rev-list --count origin/main..HEAD
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
COMMIT_COUNT=$(git rev-list --count origin/main..HEAD)
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
   git reset --soft $(git merge-base origin/main HEAD)
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

## Step 5 — Run the local quality gate

Run locally BEFORE relying on CI. Catches issues faster than waiting for GH Actions.

```bash
npm run quality:check
npm run build
```

`quality:check` runs: `lint` + `format:check` + `typecheck` + `test:threshold` (per-file coverage gate 75/65/75/70).

**Capture exit code.**

### If quality:check FAILED

Identify the failing stage from the output:

**Lint failed:**
```
❌ ESLint errors found.
Auto-fix what's fixable with `npm run lint:fix`? (y/n)
```
If yes: run autofix, stage, commit `[CHORE] Auto-fix lint - <TICKET>`, push, re-run.

**Format failed:**
```
❌ Prettier format check failed.
Run `npm run format`? (y/n)
```
If yes: run format, stage, commit `[CHORE] Apply prettier - <TICKET>`, push, re-run.

**Typecheck failed:**
```
❌ TypeScript errors. Fix manually:
  <tsc output>
```
Do NOT auto-fix. Human owns type errors.

**Coverage gate failed:**
```
❌ Per-file coverage below threshold (lines 75 / functions 65 / statements 75 / branches 70).
  <offending files>

Fix manually:
  1. Add tests covering the uncovered branches.
  2. If a branch is genuinely unreachable, mark it with /* istanbul ignore next */ and justify.
  3. Do NOT add the file to vitest.config.ts's `exclude`.
```
Do NOT auto-fix. Human writes tests.

**Build failed:**
```
❌ Vite build failed. Likely: unused imports, dynamic imports that fail tree-shaking, missing key props.
  <vite output>
```
Do NOT auto-fix. Human owns build errors.

**NEVER** move the task to Code Review if any local gate failed.

---

## Step 6 — Watch GitHub Actions CI

Once the local gate is green, CI should green too — but wait for confirmation.

```bash
gh pr checks --watch
```

The core-web-customer pipeline has 3 jobs:
1. **quality** → lint + format:check + typecheck
2. **test** → test:threshold (per-file coverage)
3. **build** → Vite production build

All 3 must pass.

### If CI failed

```bash
gh run view --log-failed
```

Report the failing step to the user. The categories match Step 5 — auto-fix lint/format, human fixes tests/types/build.

**NEVER** move the task to Code Review if any CI job failed.

---

## Step 7 — Handle success

When local + CI are both green:

1. Confirm success to the user:
   ```
   ✅ Local gate: lint ✅ format ✅ typecheck ✅ coverage ✅ build ✅
   ✅ GitHub Actions: quality ✅ test ✅ build ✅
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

Local gate (npm run quality:check && npm run build):
  Lint:       ✅
  Format:     ✅
  Typecheck:  ✅
  Coverage:   ✅ per-file 75/65/75/70
  Build:      ✅

GitHub Actions:
  quality:    ✅
  test:       ✅
  build:      ✅

Next action:
  <e.g., "PR ready for review, pinged reviewers in Slack" | "Fix failing tests and re-run">
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Hard rules

- **Always** run pre-flight validations before the quality gate — fail fast.
- **Always** squash multi-commit branches before the final push — one commit per PR.
- **Always** use `--force-with-lease`, never `--force`.
- **Never** move the task to Code Review if local gate OR any GitHub Actions job failed.
- **Never** auto-fix failing tests, type errors, or build errors — humans own those.
- **Never** bypass the coverage gate by adding files to `vitest.config.ts`'s `exclude` — write the test instead.
- **Never** disable an ESLint rule in `eslint.config.js` to silence a single-file issue — fix the code.
- **Never** push to `main` directly.
- **Never** skip reading the Notion task state — if a ticket ID is detected, check Notion.
