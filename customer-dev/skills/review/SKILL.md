---
description: "Scan the current branch's diff against main for React/TypeScript antipatterns (stale closures, missing deps, key=index, derived state, dangerouslySetInnerHTML, `any`, cross-subscription imports, unmemoized context values, missing AbortController, a11y gaps). Produces a pre-review report for the human before opening a PR. Meant for a non-React expert to catch what their eye won't."
model: sonnet
---

# React Pre-Review — core-web-customer

Scans the diff between the current branch and `main` looking for known antipatterns. Meant to run AFTER `customer-dev:execute` and BEFORE `customer-dev:pr-signoff`. Produces a report, not edits — the human decides what to fix.

**Input:** optional `$ARGUMENTS` — a subset of paths to scan. Defaults to the full diff.

---

## Step 1 — Determine diff scope

```bash
git fetch origin main --quiet
git diff --name-only origin/main...HEAD | grep -E '\.(ts|tsx)$'
```

If `$ARGUMENTS` is supplied, intersect with that list.

If empty (no changed `.ts/.tsx` files) → exit with "No TypeScript changes to review."

---

## Step 2 — Load the pitfall catalogue

Read `${CLAUDE_PLUGIN_ROOT}/context/react-pitfalls.md` in full. The scan is grounded in that list.

Also skim `${CLAUDE_PLUGIN_ROOT}/context/subscriptions.md` (for the Golden Rule) and `${CLAUDE_PLUGIN_ROOT}/context/performance.md`.

---

## Step 3 — Read each changed file

Use `Read` on each file in full. You need the whole file to reason about hooks' deps, closures, and imports.

For each file, also run:

```bash
git diff origin/main...HEAD -- <file>
```

So you see what actually changed vs. pre-existing code. Focus the scan on changed regions — don't flag pre-existing debt in files merely touched.

---

## Step 4 — Run the scans

For every changed file, look for each of the following. For each hit, note: `file:line`, the snippet, why it's wrong, and a proposed fix.

### A. TypeScript hygiene

- `\bany\b` as a type → flag.
- `Record<string, *any>` → flag.
- `as any` → flag.
- `@ts-ignore` without `// @ts-expect-error` → flag (prefer expect-error).
- `any[]` or `Array<any>` → flag.

### B. React Hooks — exhaustive-deps

- For every `useEffect`, `useMemo`, `useCallback`:
  - Read the dep array.
  - Read the body — list identifiers the body uses from the enclosing scope.
  - If any used identifier (that isn't a ref, a constant outside the component, or a setState function) is missing from the dep array → flag.
  - If the dep array contains identifiers the body doesn't use → flag (dead dep — suggests stale code).
- `// eslint-disable-next-line react-hooks/exhaustive-deps` → flag for review (usually indicates a real design issue).

### C. Stale closures in effects

- `useEffect` / `useCallback` body uses a mutable value that is NOT in deps AND is NOT being read from a ref → flag "potential stale closure."
- Interval/timeout callbacks inside an effect that reference state without the state being in deps → flag.

### D. Async effects without cleanup

- `useEffect(() => { ... fetch(...) ... })` → if no `AbortController` and no `ignore` flag, flag as "race condition risk."
- `setTimeout` / `setInterval` in an effect without a cleanup `return () => clear…` → flag.
- Event listener added (`window.addEventListener`) without matching cleanup → flag.

### E. Derived state

- `const [x, setX] = useState(props.<something>)` → flag.
- `const [x, setX] = useState(<something computed from another state>)` → flag.
- `useEffect` that calls `setX(props.y)` → flag as "reset-on-prop: prefer key or lifting state."

### F. Key antipatterns

- `.map(...).` where the rendered child has `key={i}`, `key={index}`, `key={idx}`, or a template literal `key={`${something}-${i}`}` that embeds the index → flag.
- `.map(...).` without a `key` at all on the top-level child → flag.

### G. Dangerous HTML

- `dangerouslySetInnerHTML` → flag hard. Suggest sanitizing with DOMPurify or removing.

### H. Subscription Golden Rule

For each file in `src/subscriptions/<slugA>/**`:
- Scan imports for `@/subscriptions/<slugB>/` where `slugB !== slugA`.
- Scan imports for `../../<slugB>/` / `../../../subscriptions/<slugB>/`.
- Any hit → flag as "CROSS-SUBSCRIPTION IMPORT — promote to global."

### I. `karta` touched

- Any new file under `src/subscriptions/karta/**` → flag (deprecated).
- Any import from `@/subscriptions/karta/` in new code → flag.
- Any UI code (sidebar, dashboard, route list) that adds `karta` → flag.

### J. Memoization smells

- `useCallback(fn, [])` where `fn` body references `props.*` or state without them in deps → flag (stale).
- `useMemo(() => <literal>, [...])` → flag (primitives are already stable).
- `React.memo` on a component that takes `children` or new-object props from the parent → flag as "likely ineffective."

### K. Context provider without memoized value

- `<Context.Provider value={{ a, b }}>` with a fresh object literal and no `useMemo` → flag.

### L. Page-level rule compliance (if file is `src/subscriptions/**/pages/**/index.tsx`)

Check the Index page has:
- [ ] `useRansackSearch` (not local `filter()` on data).
- [ ] `usePaginatedResource` (not a bespoke `useEffect(fetch)`).
- [ ] `didInitRef.current` guard for initial fetch.
- [ ] `setPage(1)` on `ransackQuery` change.
- [ ] `PageHeader` + `DataTable` (not a custom table).
- [ ] `ConfirmationDialog` before any delete action.
- [ ] `searchLoading={loading}` on `PageHeader`.

Flag missing items.

### M. Form-level a11y compliance (if file renders `<input>`, `<textarea>`, `<select>`)

For each control:
- [ ] Has `id` attribute.
- [ ] Has matching `<label htmlFor={id}>`.
- [ ] Has `aria-invalid` tied to error state.
- [ ] Has `aria-describedby` pointing to the error element when invalid.
- [ ] Has `required` on mandatory fields.

Flag missing items.

### N. Unhandled async in event handlers

- `onClick={async () => { await … }}` where the error isn't caught → flag (unhandled rejection will `console.error` or worse).
- Missing `try/catch` around `await` service calls in handlers → flag.

### O. Direct `fetch` usage

- `await fetch(` outside of `src/services/auth.ts` → flag ("use apiFetch").

### P. Derived / recomputed columns every render

- `<DataTable columns={...} />` where `columns` is a fresh array on every render (not memoized) → flag as perf smell.

---

## Step 5 — Run the automated checks too

Before producing the report, run these commands and capture the output — they catch things the grep scan misses:

```bash
npm run typecheck
npm run lint
```

Feed any remaining lint issues from the CHANGED files into the report as "Lint issues (automated)." Do NOT fail on issues in unchanged files.

---

## Step 6 — Write the report

Output a structured report. Group by severity. Be specific: include `file:line`, a snippet, and a suggested fix.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔍 React Pre-Review — <branch>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Scope: <N> files changed vs origin/main
  - src/subscriptions/katalog/pages/Items/index.tsx
  - src/subscriptions/katalog/services/itemsApi.ts
  - ...

━━ 🛑 BLOCKING (fix before PR) ━━

1. src/subscriptions/katalog/pages/Items/index.tsx:42
   Issue: Cross-subscription import.
     import { Resource } from '@/subscriptions/kalendar/services/resourcesApi';
   Rule: a subscription never imports from another subscription.
   Fix:  promote Resource type to src/types/ or src/services/, then import from there.

2. src/subscriptions/katalog/pages/Items/ItemsColumns.tsx:17
   Issue: key={index} in mapped row.
     {items.map((item, i) => <Row key={i} item={item} />)}
   Rule: index-as-key causes state attachment bugs on reorder.
   Fix:  key={item.id} (item has a stable id).

━━ ⚠️  LIKELY BUGS (fix or justify) ━━

3. src/subscriptions/katalog/pages/Items/index.tsx:88
   Issue: useEffect body reads `ransackQuery` but only `fetchData` is in deps.
     useEffect(() => { fetchData({ q: ransackQuery }); }, [fetchData]);
   Rule: exhaustive-deps violation → stale closure when the user types.
   Fix:  add ransackQuery to deps, OR read from a ref (ransackQueryRef.current).

4. src/subscriptions/katalog/services/itemsApi.ts:31
   Issue: raw fetch() instead of apiFetch.
     const r = await fetch(url, {...});
   Rule: authenticated endpoints must go through apiFetch (handles JWT + refresh).
   Fix:  import { apiFetch } from '@/services/api'.

5. src/subscriptions/katalog/hooks/useItemForm.ts:55
   Issue: derived state — useState(props.initialName).
   Rule: state stops tracking prop changes → Edit modal doesn't update when a new row is selected.
   Fix:  (a) lift state up; (b) <Form key={row.id} ...> so state resets; (c) useEffect reset.

━━ 💡 SUGGESTIONS (not blocking) ━━

6. src/subscriptions/katalog/pages/Items/index.tsx:12
   Issue: SEARCH_CONFIG is an object literal inside the component without useMemo.
   Impact: useRansackSearch's debounce resets every render.
   Fix: wrap in useMemo with [].

7. src/subscriptions/katalog/pages/Items/ItemsColumns.tsx:40
   Issue: action button is icon-only with no aria-label.
     <Button onClick={...}><Trash /></Button>
   Rule: screen readers announce "button" with no label.
   Fix: <Button aria-label={`Eliminar ${row.original.name}`} ...>

━━ 📊 Automated gates (from lint / typecheck) ━━

Typecheck:  PASS
Lint:       2 warnings in changed files
  - src/.../index.tsx:77 — Unused variable `foo`
  - src/.../itemsApi.ts:3 — Import ordering

━━ 📚 Pitfall references cited above ━━

- Cross-subscription import → context/subscriptions.md § "The Golden Rule"
- key=index → context/react-pitfalls.md § 3
- Exhaustive-deps violation → context/react-pitfalls.md § 1, 2
- Raw fetch → context/services.md § "apiFetch"
- Derived state → context/react-pitfalls.md § 4
- Unmemoized search config → context/hooks.md § "useRansackSearch"
- Missing aria-label → context/components.md § "a11y"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Summary: <N-blocking> blocking, <N-likely> likely bugs, <N-suggestion> suggestions.
Next:    resolve BLOCKING (any remaining) and LIKELY BUGS before `customer-dev:pr-signoff`.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

If the report is clean (no BLOCKING or LIKELY BUGS):

```
✅ No React antipatterns detected in the diff.
Automated gates:  typecheck ✅  lint ✅
Ready for:        customer-dev:pr-signoff
```

---

## Hard rules

- **Never edit code in this skill.** Report only. The human (or a follow-up execute run) fixes.
- **Scan only CHANGED code.** Do not flag pre-existing issues in a file merely because the file was touched — that creates noise and tanks signal.
- **Cite a pitfall source** for every flag. If the flag isn't traceable to a context file or an automated tool, either add it to context first or drop the flag.
- **Be specific.** `file:line`, snippet, fix. "Review this file" is not useful.
- **Err on the side of suggesting, not blocking.** "BLOCKING" is for rule violations (cross-subscription imports, `dangerouslySetInnerHTML` without sanitizer, `any` types). "LIKELY BUG" is for almost-certain runtime issues. Everything else is "SUGGESTION."
- **Finish with an automated-gate summary** — typecheck + lint results for the diff.
