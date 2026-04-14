# Quality — ESLint, Prettier, TypeScript, CI pipeline

## The CI pipeline is the source of truth

`.github/workflows/ci.yml` runs on every push to `main`/`staging` and every PR:

```
quality  ─▶  test  ─▶  build
   │          │          │
   ▼          ▼          ▼
 lint       test:      vite
 format     threshold  build
 typecheck  (coverage  (CI=true)
            gate)
```

If any job fails, CI is red. Each job installs dependencies with `npm ci`.

### Job 1: quality
```bash
npm run lint          # ESLint 9 flat config
npm run format:check  # Prettier
npm run typecheck     # tsc --noEmit -p tsconfig.app.json
```

### Job 2: test
```bash
npm run test:threshold  # vitest --coverage with per-file gates
```

### Job 3: build
```bash
CI=true npm run build   # Vite production build
```

### Local equivalent

`npm run quality:check` runs lint + typecheck + format:check + test:threshold. **It does NOT run `npm run build`.** Before pushing, run both:

```bash
npm run quality:check && npm run build
```

Otherwise your PR can pass `quality:check` locally and still fail CI on the build step (e.g., a dev-only import that breaks under Vite's production transforms).

## ESLint

Flat config (`eslint.config.js`). Plugins:
- `@typescript-eslint` — TS rules.
- `eslint-plugin-react` + `eslint-plugin-react-hooks` — React rules, including `rules-of-hooks` and `exhaustive-deps`.
- `eslint-plugin-react-refresh` — HMR rules.
- `eslint-plugin-jsx-a11y` — accessibility.
- `eslint-plugin-import` — import order/resolution.
- `eslint-config-prettier` — turns off rules that conflict with Prettier.

### Rules that matter most

| Rule | Rationale |
|---|---|
| `react-hooks/rules-of-hooks` | Hooks called conditionally → runtime error. Never silence. |
| `react-hooks/exhaustive-deps` | Missing deps → stale closures. Fix the code, not the warning. |
| `@typescript-eslint/no-explicit-any` | `any` opts out of type safety. Use generics or `unknown`. |
| `jsx-a11y/*` | Accessibility. If it complains, the UI is broken for keyboard/SR users. |
| `import/no-cycle` | Cyclic imports cause undefined-at-runtime bugs. |

### When a rule feels wrong

Don't `// eslint-disable-next-line` without a committed justification comment on the line above. If you disable the same rule 3+ times in the codebase, that's a signal to revisit the rule config, not to keep disabling.

## Prettier

- 2-space indentation.
- `npm run format` auto-fixes. `npm run format:check` gates CI.
- Husky runs `lint-staged` pre-commit (ESLint + Prettier on staged files).

## TypeScript

- Strict mode on. No `any`. No `@ts-ignore` without a justification comment.
- `tsconfig.app.json` for app code, `tsconfig.node.json` for Vite config + tooling.
- `npm run typecheck` runs `tsc --noEmit -p tsconfig.app.json`.

### Patterns that pass the linter but are still wrong

- **`as Type` assertions without verification** — prefer type guards.
- **`unknown` narrowing shortcuts** — `if ('foo' in obj)` is safer than casting.
- **`Record<string, any>`** — just as bad as `any`. Use `Record<string, unknown>` or a proper type.
- **`Function` type** — too broad. Use `(x: number) => string` or similar.

## Husky + lint-staged

Pre-commit hook runs ESLint + Prettier on staged files. If the hook fails, the commit is blocked — **do not `--no-verify`**. Fix the underlying issue.

## Build — Vite

- `vite.config.ts` sets the `/api/v1/customer` dev proxy.
- `CI=true` enables strict production mode. Common CI-only failures:
  - Unused `import` warnings becoming errors.
  - Dynamic imports that work in dev (lax) but fail tree-shaking in prod.
  - Missing `key` props in maps (lints as warning, fails with strict React).

Always run `npm run build` locally before pushing — catches issues that `typecheck` misses.

## Commit messages — Conventional Commits

```
feat: add category search with ransack predicate
fix: reset page to 1 when filter changes
refactor: extract useCategoryDelete hook
test: cover Categories error state
chore: bump @radix-ui packages
docs: update pages.mdc with StrictMode note
```

Types: `feat`, `fix`, `refactor`, `test`, `chore`, `docs`, `style`, `perf`, `ci`, `build`.

## Branch naming

```
feature/<slug>-<TICKET>      # new feature
bugfix/<slug>-<TICKET>       # bug fix
improvement/<slug>-<TICKET>  # enhancement without new feature
hotfix/<slug>-<TICKET>       # urgent prod fix
chore/<slug>-<TICKET>        # deps, config, tooling
```

## PRs

- Always target `main`.
- Include the Notion URL in the body (traceability).
- Include a test plan (what reviewer runs to verify).
- **Do not merge your own PR** without a review.
- **Do not force-push** shared branches without authorization. Use `--force-with-lease` only when it's your branch and a teammate may have pulled it.

## Dependency hygiene

- New dependencies need justification in the PR description. "It's convenient" is not enough.
- Security: `npm audit` regularly. Don't ignore high/critical.
- Pin major versions in `package.json` (already the case). Don't use `*` or `latest`.
- `depcheck` is installed — run it when suspecting unused deps.
