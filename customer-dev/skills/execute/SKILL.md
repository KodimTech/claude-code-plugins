---
description: "Execute a core-web-customer implementation plan (.md from customer-dev:plan): write tests first (RED), implement until GREEN, run lint + typecheck + test:threshold + build. Human reviews output before PR."
model: sonnet
---

# Implementation Executor — core-web-customer

You are executing a pre-approved plan produced by `customer-dev:plan`. Your job is to write correct, tested, lint-clean, type-safe code. **You do not commit. You do not push. You do not create PRs.** A human reviews everything you produce.

**Input:** `$ARGUMENTS` — path to the plan `.md` file.

---

## Step 1 — Read the plan

Read the file at `$ARGUMENTS` in full. Extract:

- **Ticket ID** + **Branch name**
- **Scope** (subscription slug or global)
- **Files to Read Before Implementing**
- **Files to Create**
- **Files to Modify**
- **Environment Variables** (if missing from `.env.example`, stop and ask)
- **Business Rules** (all of them)
- **Failure Conditions** (condition → exact message)
- **Side Effects** (refetch, toast, close modal)
- **Spec Skeleton** (per test file)
- **Implementation Steps** (ordered)

If any section is too vague to implement without guessing, **stop immediately** and tell the user. Do not proceed.

---

## Step 2 — Load context (mandatory)

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
13. `${CLAUDE_PLUGIN_ROOT}/context/rails-to-react-glossary.md`

These are the house rules. Every file must conform.

---

## Step 3 — Read existing files

Read every file listed in **"Files to Read Before Implementing"** AND **"Files to Modify"** — full content.

Skipping this step is the #1 cause of broken code. Analogues tell you the exact shape, naming, and conventions.

---

## Step 4 — Show execution plan

Before touching any file, output:

```
### Execution Plan

Branch:    <from plan>
Ticket:    <from plan>
Scope:     <from plan>

Files to create (<N>):
  - <list>

Files to modify (<N>):
  - <list with exact change>

Tests to write first (RED phase):
  - <list of test files>

Commands:
  Tests:      npm test -- <paths>
  Lint:       npm run lint -- <paths>
  Typecheck:  npm run typecheck
  Coverage:   npm run test:threshold
  Build:      npm run build
```

Proceed immediately. Do not wait for approval — the plan was already approved.

---

## Step 5 — Create branch

```bash
git checkout main
git pull origin main --quiet
git checkout -b <branch-from-plan>
```

If the branch already exists, warn and ask (switch / abort). Do not silently overwrite.

---

## Step 6 — Write tests FIRST (RED phase)

Write **every** test file from the plan's Spec Skeleton. Rules:

- **Query priority**: `getByRole` > `getByLabelText` > `getByText` > `getByTestId`.
- **`userEvent`** for interactions. `setup()` at the start of each test.
- **`findBy` / `waitFor`** for async. Never `setTimeout`.
- **4 states** covered per page/component: Loading, Error, Success, Empty.
- **Mock at the module boundary**: `vi.mock('../../services/<entity>Api')`.
- **Exact error messages** from the plan's Failure Conditions table.
- **No `any`** in test files either. Narrow with `instanceof`, `typeof`, or generics.
- **`vi.clearAllMocks()`** in `beforeEach`.
- **`<MemoryRouter>`** around components that use React Router hooks.

Run them:

```bash
npm test -- <paths from plan>
```

**Expected: ALL FAIL.** If any pass before implementation, either:
- The test is wrong (not asserting enough).
- Something already exists (check before creating).

---

## Step 7 — Implement

Follow the plan's **Implementation Steps** in order. Each step is atomic.

For each step:
1. Create/edit the files.
2. Apply ALL Business Rules.
3. Use the EXACT error messages from Failure Conditions.
4. Implement ALL side effects (refetch, toast, modal close, form reset).
5. Run specs for this step:
   ```bash
   npm test -- <paths for this step>
   ```
6. Move to next step only when current-step tests are GREEN.

### Code-writing rules (enforced on every file you produce)

**TypeScript**
- No `any`, anywhere. Use generics, `unknown` + narrowing, or discriminated unions.
- Types for entities and form data live next to the service file.
- Event handlers typed explicitly (`React.ChangeEvent<HTMLInputElement>`, `React.FormEvent<HTMLFormElement>`).
- `useRef` with `| null` for DOM refs.
- `as const` on predicate/literal configs.
- No `// @ts-ignore` without `// @ts-expect-error <reason>` preferred.

**React**
- `exhaustive-deps` respected always. No silencing.
- `useEffect` only for synchronization with external systems.
- No derived state. Compute during render or use `key`.
- `AbortController` on async effects.
- Memoize configs (`searchConfig`) and columns.
- Refs for mutable values read inside stable `useCallback`.
- `didInitRef` guard for initial fetch (StrictMode).
- `setPage(1)` on `ransackQuery` change.
- `key` always a stable unique id — never `index`.
- No `dangerouslySetInnerHTML`.

**Subscription rules**
- No imports across subscriptions. Promote to global if shared.
- No references to `karta`. If the plan accidentally touches it, stop and ask.

**Components**
- Every interactive element: accessible by role + keyboard.
- Every action button with icon-only label: `aria-label="<Action> <Row value>"`.
- Every form input: `id`, `htmlFor`, `aria-invalid`, `aria-describedby`.
- Destructive actions: `ConfirmationDialog`.
- Columns: `size` + `minSize`.
- Empty / error / loading states rendered explicitly.

**Services**
- All HTTP through `apiFetch`. No raw `fetch()`.
- Wrap bodies in Rails resource key: `{ katalog_category: { name } }`.
- `ApiException.fromResponse` on non-2xx.
- Return typed results.

**i18n**
- User-facing strings through `t()`. Add keys to `src/i18n/<slug>/*.json` as you use them.

### Failure handling (critical)

- Spec fails → read full error → attempt fix → re-run.
- **Max 3 attempts on the same failure.** Then STOP.
- Report to the user:
  - Exact failing spec + line.
  - Error output.
  - What you tried (each attempt).
  - What you believe is wrong.
  - What you need to continue.

**Never loop blindly.** Three attempts and stop.

---

## Step 8 — Full test suite for touched areas

Once every step passes individually:

```bash
npm test -- <all test paths from plan>
```

If new failures appear that weren't there in isolation, fix them — same 3-attempt rule.

---

## Step 9 — Coverage gate

```bash
npm run test:threshold
```

If any file drops below the per-file thresholds (lines 75 / statements 75 / functions 65 / branches 70):

1. **Do NOT proceed.**
2. List the offending files.
3. For each:
   - Option A: add tests covering the uncovered branches.
   - Option B: if a branch is genuinely unreachable (defensive, generated), wrap with `/* istanbul ignore next */` + a comment justifying.
4. Re-run.
5. Same 3-attempt rule.

**Do NOT add files to the `exclude` list in `vitest.config.ts`** to bypass the gate. That list is a backlog, not an escape hatch.

---

## Step 10 — Lint + format + typecheck

```bash
npm run lint
npm run format:check
npm run typecheck
```

First pass failures:
- `npm run lint:fix` for auto-fixable.
- `npm run format` for prettier.
- Fix manual issues. Do NOT disable a rule to pass the check — fix the code.

**Never modify `eslint.config.js`** to silence a rule for this feature. If a rule is wrong repo-wide, that's a separate PR with team discussion.

---

## Step 11 — Build

```bash
npm run build
```

CI runs with `CI=true`. Common CI-only failures:
- Unused imports (stricter in prod).
- Dynamic imports that work in dev but fail tree-shaking.
- Missing `key` props.

Fix any build error before declaring success. Same 3-attempt rule.

---

## Step 12 — ENV vars verification (if the plan listed any)

If the plan's Environment Variables section listed vars:

1. Verify each is referenced as `import.meta.env.VITE_*` in the code.
2. Check `.env.example` has the key (add it if missing).
3. Remind the user:
   ```
   ACTION REQUIRED (human): add the following vars to every environment (local .env, Vercel/Kamal/hosting):
     - VITE_FOO_URL
     - ...
   ```

---

## Step 13 — Final report

```
### Execution Report

Branch:   <branch>
Ticket:   <ticket>
Status:   <GREEN / PARTIAL / BLOCKED>

Files created (<N>):
  - path/to/file.tsx
  - ...

Files modified (<N>):
  - path/to/file.tsx (what changed)
  - ...

Tests:
  - <N> new test files, all passing
  - Coverage (per-file): PASS
  - Undercover-equivalent: N/A (we use Vitest perFile gate)

Quality:
  - Lint:       ✅
  - Format:     ✅
  - Typecheck:  ✅
  - Build:      ✅

ENV vars introduced: <list or "None">
i18n keys added: <list or "None">

Manual review flags:
  - <anything you had to improvise that the plan didn't specify>
  - <any Open Question the plan left unresolved>
  - <any React pitfall the reviewer should double-check>

Ready for: commit + push + PR (customer-dev:ship handles, or commit manually).
```

---

## Hard rules

- **Never commit**, never push, never open PRs.
- **Tests first**, always. RED → GREEN.
- **No `any`.** No derived state. No `key={index}`. No cross-subscription imports. No `dangerouslySetInnerHTML`.
- **`exhaustive-deps` is final authority** — if it complains, the hook is wrong.
- **Max 3 attempts** per failing spec / lint / typecheck / build. Then stop and ask.
- **Coverage bypass via `vitest.config.ts` exclude is forbidden** — write the test instead.
- **If the plan says something that conflicts with the context files**, stop and ask. Don't improvise.
- **Teach-then-code**: when writing a non-trivial React construct, include a brief "Why this works" note in the commit or in chat — 3-6 lines max.
