---
description: "Read PR review comments (Copilot + humans), triage each against the diff and the project's context files, produce a report grouped by file. On human approval: apply ACCEPT/ADAPT fixes with the 3-attempt discipline, commit, push, post replies in GitHub, and mark threads as resolved. Ignores threads already resolved."
model: sonnet
---

# Address Review Comments — core-web-customer

Reads the open review threads on the current branch's PR, evaluates each comment against the work done and the project's house rules (the 13 `context/*.md` files), and produces a triage. After a single human checkpoint, it applies the accepted fixes, commits, posts replies, and resolves the threads.

**Input:** optional `$ARGUMENTS` — PR number. If missing, infer from the current branch.

**Core principle:** each decision must cite a source — either `file:line` from the diff or `context/<file>.md § <section>`. No citation = no decision. This is what separates "addressing review comments" from "blindly following Copilot."

---

## Step 1 — Pre-flight validations

Run in parallel:

```bash
git branch --show-current
git status --porcelain
gh pr view --json number,url,title,headRefName,baseRefName,state
gh auth status
```

Validate:
- ❌ On `main` → abort.
- ❌ No PR for current branch → abort: "Create the PR first."
- ❌ Uncommitted changes → abort: "Commit or stash before running /customer-dev:address-comments." (the skill creates its own commit at the end).
- ❌ PR state is not `OPEN` → abort.
- ✅ Capture PR number, repo owner/name (from `gh repo view --json owner,name`).

---

## Step 2 — Fetch review threads (GraphQL)

Use GraphQL because the REST comments endpoints don't expose `isResolved`:

```bash
gh api graphql -f query='
query($owner: String!, $repo: String!, $number: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $number) {
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          isOutdated
          path
          line
          originalLine
          comments(first: 100) {
            nodes {
              id
              databaseId
              author { login }
              body
              createdAt
              diffHunk
              originalLine
              line
              path
            }
          }
        }
      }
    }
  }
}' -F owner=<owner> -F repo=<repo> -F number=<PR>
```

Also fetch conversation-level (issue) comments:

```bash
gh api repos/<owner>/<repo>/issues/<PR>/comments
```

### Filtering rules

For each `reviewThread`:
- **Ignore** if `isResolved: true` → the human already closed it.
- **Flag** if `isOutdated: true` → the code it commented on changed; mention this in the report but still evaluate the root concern (it may still apply to the new code).
- Otherwise → process.

Collect the **first** comment of each thread as the "ask" and any subsequent comments as the thread's conversation.

Classify authors:
- `copilot-pull-request-reviewer[bot]`, `github-copilot[bot]` → **Copilot**.
- Any other `[bot]` → **Other bot** (treat like Copilot: high bar).
- Anything else → **Human**.

---

## Step 3 — Load context (mandatory)

Read in full:

1. `${CLAUDE_PLUGIN_ROOT}/context/architecture.md`
2. `${CLAUDE_PLUGIN_ROOT}/context/subscriptions.md`
3. `${CLAUDE_PLUGIN_ROOT}/context/services.md`
4. `${CLAUDE_PLUGIN_ROOT}/context/hooks.md`
5. `${CLAUDE_PLUGIN_ROOT}/context/components.md`
6. `${CLAUDE_PLUGIN_ROOT}/context/pages.md`
7. `${CLAUDE_PLUGIN_ROOT}/context/testing.md`
8. `${CLAUDE_PLUGIN_ROOT}/context/quality.md`
9. `${CLAUDE_PLUGIN_ROOT}/context/react-mental-model.md`
10. `${CLAUDE_PLUGIN_ROOT}/context/react-pitfalls.md`
11. `${CLAUDE_PLUGIN_ROOT}/context/typescript-patterns.md`
12. `${CLAUDE_PLUGIN_ROOT}/context/performance.md`

These are the authority when deciding REJECT/ADAPT.

---

## Step 4 — Read the diff and the touched files

```bash
git fetch origin main --quiet
git diff origin/main...HEAD --stat
git diff origin/main...HEAD -- <each file with open comments>
```

For each file referenced by an open thread, `Read` the full file (you need it to evaluate the comment in context — a suggestion that looks wrong in isolation may be correct given surrounding code, and vice versa).

---

## Step 5 — Triage each open comment

For every open thread, produce a triage decision. Use exactly one of:

### ACCEPT

The comment identifies a real bug, a rule violation, or a meaningful improvement that aligns with project practices. The proposed fix is idiomatic.

Mandatory fields:
- `thread_id`
- `author_class` (Copilot / Human / Other bot)
- `path:line`
- Summary of the ask
- **Fix plan** — exact file edits.
- **Source** — `file:line` in the diff AND/OR `context/<file>.md § <section>`.
- **Reply draft** — what to post in GitHub when resolving.

### ADAPT

The comment identifies a real problem but the proposed fix is wrong, non-idiomatic, or contradicts a convention. We fix the underlying issue a different way.

Mandatory fields: same as ACCEPT, plus
- **Why not the comment's fix** — cite the context rule or project analogue.
- **Fix plan** — the idiomatic alternative.

### REJECT

The comment contradicts a project convention, a hard rule, or is a style nit already covered by Prettier/ESLint.

Mandatory fields:
- `thread_id`
- `author_class`
- `path:line`
- **Reason** — must cite either:
  - A hard rule (no `any`, Golden Rule, no `key={index}`, exhaustive-deps is final, no derived state, no `dangerouslySetInnerHTML`, no raw `fetch()`, coverage bypass forbidden, etc.), OR
  - A `context/<file>.md § <section>` that prescribes a different approach, OR
  - An automated tool (Prettier/ESLint/TS) that would have caught it — if the PR passed CI, it's not a real finding.
- **Reply draft** — polite, technical, cites the rule.

### ESCALATE

A real trade-off that needs a human decision: the reviewer is a domain owner, the ask is scope-expanding, or the context doesn't have a clear ruling.

Mandatory fields:
- `thread_id`
- `author_class`
- `path:line`
- **Why escalated** — what the ambiguity is.
- **Options** — 2–3 paths the human could pick.

### Weighting rules

- **Copilot threshold is high.** ACCEPT only if the comment pegs a real bug or a project rule. Copilot's typical output is "stylistic restructuring that looks more readable" — that's usually REJECT (redundant with our tooling) or ADAPT (real smell, wrong fix).
- **Humans default to ACCEPT/ADAPT**, unless the ask clashes with a hard rule. If it clashes, REJECT and cite the rule; don't be shy about disagreeing with a human, but do it with a source.
- **Module owners** (if you can infer from the comment — "I own this module", or repeated commenter on that path): lean toward ACCEPT/ADAPT even for subjective asks. If still rejecting, ESCALATE.

---

## Step 6 — Present the triage report

Group by file, then by line. Within a file, show Copilot and Human threads together (context is per file, not per author).

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💬 Review Comments Triage — PR #<N>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Branch:     <branch>
PR:         <PR URL>
Threads:    <open> open / <resolved> already resolved (ignored)

━━ src/subscriptions/katalog/pages/Items/index.tsx ━━

[ACCEPT] thread=<id> author=Copilot  line=42
  Ask:    "Missing key prop on the mapped Item rows."
  Source: context/react-pitfalls.md § 3 (key antipatterns)
  Fix:    add key={item.id} — item has a stable id from the API.
  Reply:  "Good catch. `key={item.id}` is the stable choice here — `item.id`
           is the Rails primary key, unique per row. Fixed in <commit>."

[REJECT] thread=<id> author=Copilot  line=88
  Ask:    "Consider extracting this useEffect into a custom hook for reusability."
  Reason: Premature abstraction. The hook isn't reused anywhere (grep shows
          1 call site). Our CLAUDE rules say: "Three similar lines is better
          than a premature abstraction."
  Reply:  "Keeping this inline for now — the effect isn't reused anywhere else,
           and extracting would add a file without callers. Per project
           conventions we extract hooks only when there's a second call site."

[ADAPT]  thread=<id> author=@juan    line=55
  Ask:    "Use useMemo to avoid recomputing this on every render."
  Real problem: object literal passed as hook dep causes effect loop (diagnosed
                at context/react-pitfalls.md § 6).
  Why not the proposed fix: useMemo on a 2-item array isn't what helps here —
                            the loop comes from useRansackSearch's SEARCH_CONFIG,
                            not this computation. See context/hooks.md §
                            useRansackSearch.
  Fix:    wrap SEARCH_CONFIG in useMemo with [] deps.
  Reply:  "Thanks, this pointed at a real bug but in a different place — the
           loop comes from SEARCH_CONFIG not being memoized, not this line.
           Fixed by memoizing SEARCH_CONFIG (see context/hooks.md)."

[ESCALATE] thread=<id> author=@ana    line=120
  Ask:    "Can we batch these two API calls into one endpoint?"
  Why:    That's a backend change in core-web-api. Out of scope for this PR.
  Options:
    A) Defer to a new ticket, reply "out of scope, filed as INGEST-123"
    B) Combine client-side with Promise.all (no backend change)
    C) Scope-expand this PR with a backend endpoint (needs api-dev work)

━━ src/subscriptions/katalog/services/itemsApi.ts ━━

[ACCEPT] ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Summary:
  ACCEPT:   <N>   → will be implemented
  ADAPT:    <N>   → will be implemented (different fix than proposed)
  REJECT:   <N>   → reply will explain, no code change
  ESCALATE: <N>   → no action, waiting for your decision

Outdated threads (code has changed since the comment): <N>
  - <list>
```

---

## ⏸️ CHECKPOINT — Human approval

Ask:

```
What next?
  1. Proceed with the triage as-is (apply ACCEPT/ADAPT, post replies for REJECT, leave ESCALATE for you)
  2. Edit the triage (I'll open a .md for you to reassign categories)
  3. Skip ESCALATE handling this run (proceed with 1 but leave escalated threads untouched)
  4. Cancel

Your choice:
```

### Handling

- **1** → Step 7.
- **2** → write `pr-<N>-triage.md` with the full report, run `$EDITOR` on it, re-parse after save, then re-ask.
- **3** → same as 1 but skip ESCALATE threads entirely (no post, no change). They remain open for the human.
- **4** → exit. No files touched, no replies posted.

---

## Step 7 — Apply ACCEPT and ADAPT fixes

For each approved ACCEPT/ADAPT, in order (group by file to minimize re-reads):

1. Edit the file as the triage specifies.
2. If the change is logic-bearing (not a rename, comment, or a11y attribute), check whether there's a test covering it:
   - If yes → run it.
   - If no and the triage flagged a bug → write a test first (RED), then implement (GREEN).
3. Run tests for touched files:
   ```bash
   npm test -- <paths>
   ```
4. **3-attempt rule**: if a fix fails 3 times, STOP. Report what was applied, what's pending, and what went wrong. Do NOT post replies for unapplied fixes.

After all fixes:

```bash
npm run lint -- <touched files>
npm run typecheck
```

Auto-fix lint with `npm run lint:fix` if safe. Typecheck errors → STOP and report.

---

## Step 8 — Commit and push

Build the commit message from the triage:

```
[FIX] Address PR review comments - <TICKET>

- <thread summary 1> (<author_class>, <path>:<line>)
- <thread summary 2> (<author_class>, <path>:<line>)
- ...

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
```

```bash
git add <touched files>
git commit -m "$(cat <<'EOF'
<message above>
EOF
)"
git push
```

If the PR was previously squashed and pushed with `--force-with-lease` (by `pr-ready`), the new commit appends normally — no force-push needed here.

---

## Step 9 — Post replies and resolve threads

For each thread that had a decision in the triage (ACCEPT, ADAPT, or REJECT — NOT ESCALATE):

### Post the reply

Use the REST replies endpoint (works for inline review comments):

```bash
gh api \
  --method POST \
  repos/<owner>/<repo>/pulls/<PR>/comments/<databaseId>/replies \
  -f body="<reply draft from triage>"
```

For conversation-level (issue) comments that don't have a `databaseId` in a review thread, post a top-level issue comment instead:

```bash
gh api \
  --method POST \
  repos/<owner>/<repo>/issues/<PR>/comments \
  -f body="<reply>"
```

### Resolve the thread

Use the GraphQL mutation:

```bash
gh api graphql -f query='
mutation($threadId: ID!) {
  resolveReviewThread(input: { threadId: $threadId }) {
    thread { id isResolved }
  }
}' -F threadId=<thread.id>
```

Verify the response shows `isResolved: true`. If not, log the failure but don't abort — just report which threads couldn't be resolved.

### Order matters

Post the reply **before** resolving the thread, so the reviewer sees the response rendered in the resolved state. GitHub also sends a single notification when you resolve-after-reply.

### ESCALATE threads

Do **not** post, do **not** resolve. They stay open, the human handles them manually.

---

## Step 10 — Final report

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Address Comments — PR #<N>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Applied:
  ACCEPT:   <N> fixes committed (<commit SHA>)
  ADAPT:    <N> fixes committed (<commit SHA>)

Replied + resolved:
  <N> threads replied and marked resolved

Left open for you:
  ESCALATE:  <N> threads — see report above for options
  Unapplied: <N> threads (3-attempt rule triggered) — see STOP report

Quality after fixes:
  Tests:     <N> passing (<touched files>)
  Lint:      ✅
  Typecheck: ✅

Push:        <pushed to origin/<branch>>
PR:          <PR URL>

Next:
  - Review ESCALATE items and pick an option
  - Re-run /customer-dev:address-comments if reviewers post new comments
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Hard rules

- **Ignore resolved threads.** `isResolved: true` means the human already closed it — don't re-open, don't comment, don't evaluate.
- **Every decision cites a source.** Diff line, context file + section, or a hard rule. No citation = the decision is not valid; either find the source or ESCALATE.
- **Copilot threshold is high.** Don't ACCEPT stylistic restructures that duplicate ESLint/Prettier. That's what CI is for.
- **Never auto-apply without the checkpoint.** The human must see the triage before any file is edited or any reply is posted.
- **Never resolve a thread you didn't reply to.** The reply-then-resolve order is mandatory.
- **Never reply on behalf of the user for an ESCALATE.** Those are for humans.
- **3-attempt rule** on applying fixes. Same as `execute`. Stop and report.
- **Never skip the lint/typecheck after fixes.** A rushed fix that breaks the build is worse than no fix.
- **If the PR has been squashed since the last reviewer comment** (triage-relevant code moved to a new commit SHA), check the comments are still on the latest commit's lines — GitHub flags these as `isOutdated`. Re-evaluate them against the current file, not the commented line.

---

## When NOT to use `/customer-dev:address-comments`

- No reviewer has commented yet → nothing to triage.
- You're still iterating on the implementation (use `/customer-dev:execute` or `/customer-dev:review`).
- You want to reply manually to a single tricky comment → just do it in GitHub. This skill is for batching.
- The comments are on `main` commits (post-merge) → this skill only handles open PRs.
