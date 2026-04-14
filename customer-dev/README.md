# customer-dev

Claude Code plugin for the **core-web-customer** team at Kodim. Acts as a **Senior ReactJS + TypeScript Developer**: takes a Notion task, generates a detailed `.md` implementation plan, executes it by writing tests (RED → GREEN) and code against the project's house rules, scans the diff for React antipatterns before you open a PR, and walks the PR through signoff.

Designed for backend-minded developers working on the frontend: the context files include a React mental model, a catalog of pitfalls, TypeScript patterns, a performance guide, and a Rails→React glossary so that someone fluent in core-web-api can work productively in core-web-customer without reinventing conventions or tripping on hooks semantics.

**Flows:**

### End-to-end (recommended for new tasks)
```
/customer-dev:ship <notion-url>   → [Opus → Sonnet → Sonnet → Sonnet]
                                     plan → ⏸️ approve → execute → review → ⏸️ approve → commit → PR → signoff
```

Two human checkpoints (plan approval, commit approval). A React antipattern scan runs automatically between execute and the commit checkpoint. Everything else is automated.

### Individual skills (for targeted use cases)
```
/customer-dev:plan <notion-url>                    → [Opus]   generates plan-<TICKET>-<slug>.md
/customer-dev:execute plan-<TICKET>-<slug>.md      → [Sonnet] writes tests (RED) → implements → coverage → lint → typecheck → build → report
/customer-dev:review                               → [Sonnet] scans the branch diff for React/TS antipatterns, produces a BLOCKING / LIKELY BUGS / SUGGESTIONS report
/customer-dev:pr-signoff [<notion-url>]            → [Sonnet] squash → quality:check → build → watch GH Actions → update Notion
```

A human reviews everything before code is committed. **Skills never commit, push, or open PRs on their own — only `customer-dev:ship` does, and only after the human approves at checkpoint 2.**

### Model choice (enforced by the skills)

- **`customer-dev:ship`** runs on **Opus** — orchestrator with reasoning for checkpoints.
- **`customer-dev:plan`** runs on **Opus** — complex reasoning: classifying the subscription, designing tests with exact values, anchoring analogues with `path:line`, detecting subtle ENV vars and i18n keys.
- **`customer-dev:execute`** runs on **Sonnet** — more mechanical: follow the plan, write RED tests, implement until GREEN, run lint/typecheck/build.
- **`customer-dev:review`** runs on **Sonnet** — focused grep-style scan with pitfall citations.
- **`customer-dev:pr-signoff`** runs on **Sonnet** — deterministic git/CI flow.

All skills declare `model:` in their frontmatter, so they run on the right model regardless of your session's active model. Internal skill invocations from `customer-dev:ship` each respect their own `model:`.

---

## Automatic hooks

The plugin registers three PostToolUse hooks in `settings.json` that fire automatically while you work in core-web-customer (requires `jq` installed):

1. **Cross-subscription import guard** — when you Edit/Write a file under `src/subscriptions/<slugA>/**`, warns if the file imports from `@/subscriptions/<slugB>/` where `slugB !== slugA`. This is the single most common rule break in the codebase.
2. **React smell detector** — when you Edit/Write any `.ts`/`.tsx` file, greps for `any` types, `dangerouslySetInnerHTML`, `eslint-disable-next-line react-hooks/exhaustive-deps`, and `key={index}` patterns. Prints a warning per hit.
3. **Push pipeline reminder** — when you run `git push`, reminds you that GitHub Actions will run `quality` + `test` + `build` on push and the coverage gate is per-file (75/65/75/70), not global.

These are **non-blocking** (stderr warnings only). They exist to cut down on three common mistakes.

---

## What makes this plugin specific to core-web-customer

This plugin understands the unique patterns of **core-web-customer**:

- **Multi-subscription SPA** with a Golden Rule: each subscription (`kathy`, `katalog`, `kalendar`) is isolated — no cross-imports. Shared code is promoted to `src/` globals.
- **`karta` is deprecated** — the folder exists but nothing should render it. The agent, hooks, and review skill flag any new reference.
- **`useRansackSearch` + `usePaginatedResource`** — the canonical server-side search + pagination pair. Plans and executions anchor against the reference `Categories/index.tsx` page.
- **Kalendar endpoint aliasing** — "Servicios" hits `/kalendar/resources` and "Tipos de Servicio" hits `/kalendar/resource_types`. The plan and services context capture this so the agent doesn't guess.
- **`apiFetch` + `ApiException.fromResponse`** — every authenticated HTTP call goes through `src/services/auth.ts`. The review skill flags raw `fetch()` outside that file.
- **Rails nested attributes** — services wrap bodies like `{ katalog_category: { ..., items_attributes: [{ _destroy: true }] } }`.
- **Per-file coverage gate** — `vitest.config.ts` enforces `perFile: true` with thresholds `lines 75 / functions 65 / statements 75 / branches 70`. Bypassing via `exclude` is forbidden.
- **3-job GitHub Actions pipeline** — `quality` (lint + format:check + typecheck), `test` (test:threshold), `build`. All three gate the PR.
- **React 18 StrictMode double-invocation** — the `didInitRef.current` guard pattern is mandatory for initial-fetch effects.
- **i18n via i18next** — user-facing strings go through `t()`, keys live in `src/i18n/<slug>/*.json`, added as you use them.
- **shadcn + Radix + Tailwind** — primitives are composed, not forked. The components context spells out the a11y wiring (`htmlFor`, `aria-invalid`, `aria-describedby`).

See `context/*.md` for the full rules embedded in the plugin.

### Teaching layer (for Rails developers)

Because core-web-customer is a React codebase and the team is backend-heavy, this plugin ships with extra teaching material you won't find in `api-dev`:

- `context/react-mental-model.md` — render cycle, closures, batching, effect lifecycle, when-and-why memoization.
- `context/react-pitfalls.md` — 20 numbered antipatterns with smell + fix + why it happens.
- `context/typescript-patterns.md` — event handlers, generics, discriminated unions, narrowing, `as const` / `satisfies`.
- `context/performance.md` — measure-first guidance for when memoization actually helps.
- `context/rails-to-react-glossary.md` — one-to-one mapping tables (MVC split, request lifecycle, forms, testing, deployment).

The Senior React agent has a **teach-then-code** directive: on the first non-trivial React construct in a session, it writes 3–6 lines of "why this works" alongside the code. The `review` skill grounds every flag in a specific context file so you can follow the link and learn the rule.

---

## Requirements

- [Claude Code](https://claude.ai/code) installed
- Notion MCP configured (the plugin uses it to read tasks and update statuses)
- `gh` CLI installed and authenticated (`gh auth login`)
- Node 20+ and `npm` (matches the CI pipeline)
- `jq` installed (the hooks use it to parse tool call inputs)

---

## Installation

### 1. Register the marketplace

From inside Claude Code, run:

```
/plugin marketplace add KodimTech/claude-code-plugins
```

This registers the `kodim-api` marketplace in your local config. Only needed once.

### 2. Install the plugin

```
claude plugin install customer-dev@kodim-api
```

### 3. Enable it

```
claude plugin enable customer-dev@kodim-api
```

### 4. Reload

```
/reload-plugins
```

You should see `customer-dev` listed in the plugins count.

---

## Usage

### Generate a plan

```
/customer-dev:plan https://www.notion.so/workspace/task-title-abc123
```

Or with the raw page ID:

```
/customer-dev:plan abc123...
```

The planner:
1. Fetches task details from Notion (title, description, acceptance criteria, ticket, ENV vars mentioned)
2. Loads the 13 context files
3. Explores the core-web-customer codebase to anchor the plan in real analogues (the reference `Categories/index.tsx` page, its service, its hooks)
4. Shows a visible analysis before writing
5. Validates ENV vars against `.env.example`
6. Writes `plan-<TICKET>-<slug>.md` in your current directory
7. Stops and waits — no branch, no files, no commits

### Execute a plan

```
/customer-dev:execute plan-KODIM-456-add-item-variants.md
```

The executor:
1. Reads the plan (if any section is vague, it stops and asks)
2. Loads the 13 context files
3. Reads every analogue and modified file listed in the plan
4. Shows an execution plan before touching anything
5. Creates the branch from `main`
6. Writes every test from the Spec Skeleton first — **RED phase** (all must fail)
7. Implements the plan step by step, running tests after each step
8. Max 3 attempts per failing test, then stops and asks
9. Runs full test suite + `npm run test:threshold` (per-file coverage, blocking gate)
10. Runs `npm run lint && npm run format:check && npm run typecheck`
11. Runs `npm run build` (catches CI-only failures like unused imports)
12. Documents any new ENV vars and adds them to `.env.example`
13. Produces a final report with exact commit/push/PR commands for the human

### Scan the diff for React antipatterns

```
/customer-dev:review
```

The reviewer:
1. Diffs the current branch against `origin/main`, filtered to `.ts`/`.tsx`
2. Loads the pitfall catalogue from `context/react-pitfalls.md` and friends
3. Reads each changed file in full and scans for 16 categories of smells (cross-subscription imports, `any` types, `key={index}`, stale closures, missing effect cleanup, derived state, unmemoized context values, raw `fetch()`, form a11y gaps, `karta` references, …)
4. Runs `npm run typecheck` + `npm run lint` to capture automated findings for the changed files
5. Produces a report grouped by **BLOCKING** / **LIKELY BUGS** / **SUGGESTIONS** — each flag includes `file:line`, snippet, why, proposed fix, and a link to the exact context section
6. Does **not** edit code — the human (or a follow-up `execute` run) fixes

### PR signoff

```
/customer-dev:pr-signoff [<notion-url>]
```

The signoff:
1. Validates branch naming, PR existence, clean working tree
2. Extracts the ticket ID from the branch (or the argument)
3. Squashes multi-commit branches with `--force-with-lease`
4. Verifies the Notion task state
5. Runs the local gate: `npm run quality:check && npm run build`
6. Watches GitHub Actions via `gh pr checks --watch` for all 3 jobs green
7. Moves the Notion task to `Code Review` on success

---

## Update

```
claude plugin update customer-dev@kodim-api --scope local
```

If it errors with scope issues:

```
claude plugin uninstall customer-dev@kodim-api
claude plugin install customer-dev@kodim-api
```

---

## Structure

```
customer-dev/
├── .claude-plugin/
│   └── plugin.json
├── agents/
│   └── senior-react-dev.md         # Senior React + TS persona + rules (Opus)
├── context/
│   ├── architecture.md             # App shell, routing, auth, i18n, theming, build
│   ├── subscriptions.md            # Golden Rule + promotion checklist
│   ├── services.md                 # apiFetch, Ransack, Pagy, nested attributes, Kalendar aliases
│   ├── hooks.md                    # useRansackSearch, usePaginatedResource, StrictMode-safe init
│   ├── components.md               # shadcn/Radix primitives + forms + tables + a11y
│   ├── pages.md                    # CRUD Index page skeleton (reference: Categories/index.tsx)
│   ├── testing.md                  # Vitest + RTL + 4 states (Loading/Error/Success/Empty)
│   ├── quality.md                  # CI pipeline, ESLint/Prettier/TS, Husky, commits, branches
│   ├── react-mental-model.md       # Render cycle, closures, batching, effect lifecycle
│   ├── react-pitfalls.md           # 20 numbered antipatterns
│   ├── typescript-patterns.md      # Generics, discriminated unions, narrowing, as const / satisfies
│   ├── performance.md              # Measure-first memoization guidance
│   └── rails-to-react-glossary.md  # Mapping tables for Rails developers
├── skills/
│   ├── ship/
│   │   └── SKILL.md                # End-to-end orchestrator (plan → execute → review → commit → signoff)
│   ├── plan/
│   │   └── SKILL.md                # Planner workflow
│   ├── execute/
│   │   └── SKILL.md                # Executor workflow (TDD + coverage gate)
│   ├── review/
│   │   └── SKILL.md                # React antipattern scanner (diff-scoped)
│   └── pr-signoff/
│       └── SKILL.md                # quality:check + GH Actions watch + Notion status update
├── settings.json                   # Activates the senior-react-dev agent + 3 hooks
└── README.md
```

---

## Hard rules the plugin enforces

- **Tests first**: RED tests before implementation, then GREEN. No exceptions.
- **No `any`**: generics, `unknown` + narrowing, or discriminated unions. Review flags `any`, `as any`, `any[]`.
- **No `key={index}`**: a stable unique id, always. Review flags index keys.
- **No derived state**: compute during render or use `key` to reset. Review flags `useState(props.x)`.
- **No cross-subscription imports**: review + automatic hook both flag `@/subscriptions/<other>/`.
- **No `dangerouslySetInnerHTML`** without sanitization: review flags hard.
- **Per-file coverage gate (75/65/75/70)**: adding to `vitest.config.ts` `exclude` is forbidden — write the test.
- **`exhaustive-deps` is final authority**: no silencing with `eslint-disable-next-line`.
- **3-failure rule**: max 3 attempts per failing test / lint / typecheck / build, then stop and ask.
- **Never commits, pushes, or opens PRs** — the human owns the Git surface; `ship` is the only skill that does, and only after Checkpoint 2.
