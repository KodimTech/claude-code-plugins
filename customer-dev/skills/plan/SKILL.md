---
description: Fetch a Notion task by URL or ID, explore core-web-customer codebase, and generate a high-detail spec-driven implementation plan (.md) ready for customer-dev:execute.
model: opus
---

# Implementation Planner — core-web-customer

## Input

**$ARGUMENTS** — Notion URL (e.g., `https://www.notion.so/workspace/Task-Title-abc123...`) or a Notion `page_id` (32-char hex).

If missing, abort with:
```
Usage: /customer-dev:plan <notion-url-or-page-id>
```

---

## Step 1 — Validate environment

Run in parallel:
```bash
git branch --show-current
git status --porcelain
git fetch origin main --quiet
```

- If uncommitted changes → warn. Ask to stash or continue.
- Inform current branch.

---

## Step 2 — Fetch the Notion task

Extract the page ID (last 32-char hex segment of URL) or use the ID directly.

Call:
1. `mcp__notion__API-retrieve-a-page` → properties (title, status, ticket).
2. `mcp__notion__API-get-block-children` → body (description, acceptance criteria, any tables).

Extract fields (Notion template uses **Spanish** headings):

| Field | Source |
|---|---|
| Title | `properties.title` or first `heading_1` |
| Description | Paragraphs under `Descripción` heading |
| Acceptance Criteria | `bulleted_list_item` or `to_do` under `Criterios de aceptación` |
| Ticket ID | Property or regex on title (`KODIM-\d+`, `SC-\d+`) |

**Also extract**:
- **Subscription hint** — mentions of `kathy`, `katalog`, `kalendar`, `karta` (deprecated — flag if referenced), or "global" (users, onboarding, dashboard).
- **New env vars** — `VITE_*` identifiers, phrases like "nueva clave", "variable de entorno".
- **New endpoint mentions** — if the task needs a backend endpoint that doesn't exist yet → Open Question.

---

## Step 3 — Load context (mandatory)

Read these files IN FULL:

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

Non-negotiable: the plan must align with these files. Conflicts → flag in "Open Questions."

---

## Step 4 — Explore the actual codebase

For each component inferred from the task, find an analogous file using `Glob` + `Grep` + `Read`:

| Component | Glob pattern |
|---|---|
| Subscription entry | `src/subscriptions/<slug>/<Subscription>.tsx` |
| Index page | `src/subscriptions/<slug>/pages/<Entity>/index.tsx` |
| Columns file | `src/subscriptions/<slug>/pages/<Entity>/<Entity>Columns.tsx` |
| Create modal | `src/subscriptions/<slug>/pages/<Entity>/Create<Entity>Modal.tsx` |
| Service | `src/subscriptions/<slug>/services/<entity>Api.ts` |
| Form hook | `src/subscriptions/<slug>/hooks/use<Entity>Form.ts` |
| Delete hook | `src/subscriptions/<slug>/hooks/use<Entity>Delete.ts` |
| Global hook | `src/hooks/use<Thing>.ts` |
| Global component | `src/components/<Thing>.tsx` |
| Route table | `src/App.tsx` |
| Tests | next to the file, inside `__tests__/` |

**Every file in "Files to Create" must reference an analogue with `path:line`.** If no analogue exists, use a cross-subscription analogue and note it.

Read 1-2 of the most relevant analogues in full — they anchor the plan in real code.

---

## Step 5 — Show analysis (visible, not internal)

Before writing the plan file, output this block:

```
### Analysis

- Title:              <Notion title>
- Ticket:             <KODIM-XXX | ⚠️ NOT SPECIFIED>
- Type:               [feature | bugfix | improvement | chore | test]
- Scope:              [subscription: kathy | katalog | kalendar | global]
- Entity:             <e.g., Category, Booking, User>
- Components:         [Service, Hook, Page, Form, Modal, Column defs, Route, Global component, Context, …]
- Pattern chosen:     [e.g., CRUD Index page + Create modal + 2 subscription hooks]
- Why:                [1-2 sentences]
- Lazy-loaded?:       [route under an existing lazy subscription / new lazy segment]
- New env vars:       [list or "None"]
- Backend dependency: [new endpoint needed? if yes, link to the api task in Notion]
- Risks:              [anything ambiguous, backend uncertainty, a11y concerns]
- Cross-subscription: [any temptation to share code between subscriptions? document promotion plan]
```

If there are critical open questions (missing acceptance criteria, backend endpoint undefined, deprecated subscription `karta` referenced, ambiguous scope), **stop here** and list them. Don't generate the plan — ask the user.

---

## Step 6 — Write the plan

Create `plan-<ticket>-<slug>.md` in the current directory. If no ticket, use the slug with a `⚠️` prefix.

> **Quality bar**: the plan is consumed by `customer-dev:execute`. Every section must be precise. Vague → wrong code.

---

### Plan template

````markdown
# Plan: <Title>

**Ticket:** <KODIM-XXX | ⚠️ NOT SPECIFIED>
**Source:** <Notion URL>
**Branch:** `<feature|bugfix|improvement|chore>/<slug>-<TICKET>`
**Type:** Feature | Bug | Chore | Refactor
**Scope:** <subscription slug | global>
**Complexity:** Low | Medium | High

---

## Summary

<2-3 sentences synthesizing the technical problem. Do not copy the Notion description.>

---

## Acceptance Criteria (from Notion)

- [ ] <criterion 1>
- [ ] <criterion 2>
- ...

---

## Technical Decision Record

- **Pattern:** <e.g., CRUD Index + Create modal / new global hook / new route under existing lazy subscription>
- **Why:** <1-2 sentences>
- **Subscription:** <slug | "global — not in any subscription">
- **Backend dependency:** <list of endpoints the backend must expose, with status — already exists / being added in ticket XXX / not yet requested>
- **Trade-offs / risks:** <honest list>

---

## Files to Read Before Implementing

| File | Why |
|---|---|
| `src/subscriptions/<slug>/pages/<Existing>/index.tsx` | Reference Index page shape |
| `src/subscriptions/<slug>/services/<existing>Api.ts` | Reference service + types |
| `src/subscriptions/<slug>/hooks/use<Existing>Form.ts` | Reference form hook |
| `src/hooks/useRansackSearch.ts` | Understand debounce + query-shape |
| `src/hooks/usePaginatedResource.ts` | Understand pagination refs |
| `src/components/data-table/*.tsx` | DataTable API (props) |
| `src/components/PageHeader.tsx` | Page header API |

---

## Files to Create

| File | Analogue |
|---|---|
| `src/subscriptions/<slug>/services/<entity>Api.ts` | `src/subscriptions/<slug>/services/<similar>Api.ts:1` |
| `src/subscriptions/<slug>/hooks/use<Entity>Form.ts` | `src/subscriptions/<slug>/hooks/use<Similar>Form.ts:1` |
| `src/subscriptions/<slug>/hooks/use<Entity>Delete.ts` | `src/subscriptions/<slug>/hooks/use<Similar>Delete.ts:1` |
| `src/subscriptions/<slug>/pages/<Entity>/index.tsx` | `src/subscriptions/<slug>/pages/<Similar>/index.tsx:1` |
| `src/subscriptions/<slug>/pages/<Entity>/<Entity>Columns.tsx` | `src/subscriptions/<slug>/pages/<Similar>/<Similar>Columns.tsx:1` |
| `src/subscriptions/<slug>/pages/<Entity>/Create<Entity>Modal.tsx` | `src/subscriptions/<slug>/pages/<Similar>/Create<Similar>Modal.tsx:1` |
| `src/subscriptions/<slug>/services/__tests__/<entity>Api.test.ts` | analogue path |
| `src/subscriptions/<slug>/hooks/__tests__/use<Entity>Form.test.ts` | analogue path |
| `src/subscriptions/<slug>/pages/<Entity>/__tests__/<Entity>.test.tsx` | analogue path |

---

## Files to Modify

| File | Exact change |
|---|---|
| `src/App.tsx` | Add lazy route `/<slug>/<entity>` inside the existing subscription block |
| `src/subscriptions/<slug>/<Subscription>.tsx` | Add `<Route path="<entity>" element={<Entity />} />` |
| `src/config/kodimProducts.ts` | Update if adding a product (not when adding a sub-page) |
| `src/i18n/<slug>/<lang>.json` | Add translation keys used in the new page |

---

## Environment Variables

<If none: write "No new environment variables introduced." and skip the table.>

| Variable | Scope | Usage | Example |
|---|---|---|---|
| `VITE_FOO_URL` | build-time | `import.meta.env.VITE_FOO_URL` | `https://api.example.com` |

**Action required (human)**: add to `.env`, `.env.example`, and any environment (Vercel / Kamal / etc.) the app deploys to.

---

## Business Rules

Exhaustive — if not listed, not implemented.

- Rule 1: <e.g., Search is server-side via Ransack with `cont` predicate on `name` + `email`>
- Rule 2: <e.g., Page resets to 1 when the search input changes>
- Rule 3: <e.g., Delete action requires ConfirmationDialog>

---

## Failure Conditions

Every error path with the **exact** message string the user will see.

| Condition | Surface | Message |
|---|---|---|
| API returns 422 | `toast.error` | `err.message` from `ApiException.fromResponse` |
| API returns 404 | `toast.error` | `"No encontramos el recurso solicitado."` |
| Network failure | `toast.error` | `"Error de conexión. Intentá de nuevo."` |
| Form validation | `<span id="…-error" role="alert">` | Per-field message |

---

## Side Effects (on success)

- Refetch list (`refetch()`).
- Toast success with entity name: `"<Entity> creada"`.
- Close modal.
- Reset form state.

---

## Spec Skeleton

> Executor writes specs **first**, verifies RED, then implements until GREEN.
> **4 states per component/page**: Loading, Error, Success, Empty.
> **Prefer role queries**. `userEvent` over `fireEvent`. `findBy*` / `waitFor` for async.

### `src/subscriptions/<slug>/services/__tests__/<entity>Api.test.ts`

```ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { searchCategories, createCategory } from '../categoriesApi';

vi.mock('@/services/api', async () => ({
  ...(await vi.importActual('@/services/api')),
  apiFetch: vi.fn(),
  API_BASE_URL: '/api/v1/customer',
}));

import { apiFetch } from '@/services/api';
const mocked = vi.mocked(apiFetch);

describe('categoriesApi', () => {
  beforeEach(() => vi.clearAllMocks());

  describe('searchCategories', () => {
    it('POSTs to /katalog/categories/search with the ransack query', async () => {
      mocked.mockResolvedValue(
        new Response(JSON.stringify([{ id: 1, name: 'Pizza' }]), { status: 200 })
      );
      const result = await searchCategories({ ransackQuery: { q: { name_cont: 'Piz' } } });
      expect(mocked).toHaveBeenCalledWith(
        expect.stringContaining('/katalog/categories/search'),
        expect.objectContaining({
          method: 'POST',
          body: expect.stringContaining('"name_cont":"Piz"'),
        })
      );
      expect(result).toEqual([{ id: 1, name: 'Pizza' }]);
    });

    it('throws ApiException on non-2xx', async () => {
      mocked.mockResolvedValue(new Response('{"message":"bad"}', { status: 422 }));
      await expect(searchCategories()).rejects.toThrow();
    });
  });

  describe('createCategory', () => {
    it('wraps the payload under katalog_category', async () => {
      mocked.mockResolvedValue(new Response(JSON.stringify({ id: 2, name: 'New' }), { status: 201 }));
      await createCategory({ name: 'New' });
      expect(mocked).toHaveBeenCalledWith(
        expect.any(String),
        expect.objectContaining({
          body: JSON.stringify({ katalog_category: { name: 'New' } }),
        })
      );
    });
  });
});
```

### `src/subscriptions/<slug>/hooks/__tests__/use<Entity>Form.test.ts`

```ts
import { renderHook, act, waitFor } from '@testing-library/react';
import { describe, it, expect, vi, beforeEach } from 'vitest';

vi.mock('../../services/categoriesApi', () => ({
  createCategory: vi.fn(),
  updateCategory: vi.fn(),
}));

import { useCategoryForm } from '../useCategoryForm';
import * as api from '../../services/categoriesApi';
const mocked = vi.mocked(api);

describe('useCategoryForm', () => {
  beforeEach(() => vi.clearAllMocks());

  it('creates a category and fires onSuccess', async () => {
    const onSuccess = vi.fn();
    mocked.createCategory.mockResolvedValue({ id: 1, name: 'X' });
    const { result } = renderHook(() => useCategoryForm({ onSuccess }));
    act(() => { result.current.setName('X'); });
    await act(async () => { await result.current.submit(); });
    expect(mocked.createCategory).toHaveBeenCalledWith({ name: 'X' });
    expect(onSuccess).toHaveBeenCalled();
  });

  it('reports error via onActionStateChange on failure', async () => {
    const onActionStateChange = vi.fn();
    mocked.createCategory.mockRejectedValue(new Error('boom'));
    const { result } = renderHook(() => useCategoryForm({ onActionStateChange }));
    act(() => { result.current.setName('X'); });
    await act(async () => { await result.current.submit().catch(() => {}); });
    expect(onActionStateChange).toHaveBeenCalledWith(true);
    expect(onActionStateChange).toHaveBeenLastCalledWith(false);
  });
});
```

### `src/subscriptions/<slug>/pages/<Entity>/__tests__/<Entity>.test.tsx`

```tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { MemoryRouter } from 'react-router-dom';
import { describe, it, expect, vi, beforeEach } from 'vitest';

vi.mock('../../services/categoriesApi', () => ({
  searchCategories: vi.fn(),
  createCategory: vi.fn(),
  updateCategory: vi.fn(),
  deleteCategory: vi.fn(),
}));

import { Categories } from '../index';
import * as api from '../../services/categoriesApi';
const mocked = vi.mocked(api);

const renderPage = () =>
  render(
    <MemoryRouter initialEntries={['/katalog/categories']}>
      <Categories />
    </MemoryRouter>
  );

describe('Categories', () => {
  beforeEach(() => {
    vi.clearAllMocks();
    mocked.searchCategories.mockResolvedValue([{ id: 1, name: 'Pizza' }]);
  });

  it('shows loading, then the list', async () => {
    renderPage();
    expect(screen.getByRole('status')).toBeInTheDocument();
    expect(await screen.findByText('Pizza')).toBeInTheDocument();
  });

  it('shows the empty state when the list is empty', async () => {
    mocked.searchCategories.mockResolvedValue([]);
    renderPage();
    expect(await screen.findByText(/no hay categorías/i)).toBeInTheDocument();
  });

  it('shows an error state when fetch fails', async () => {
    mocked.searchCategories.mockRejectedValue(new Error('boom'));
    renderPage();
    expect(await screen.findByText(/error/i)).toBeInTheDocument();
  });

  it('opens the create modal and creates a category', async () => {
    mocked.createCategory.mockResolvedValue({ id: 2, name: 'New' });
    const user = userEvent.setup();
    renderPage();
    await user.click(await screen.findByRole('button', { name: /nueva categoría/i }));
    await user.type(screen.getByLabelText(/nombre/i), 'New');
    await user.click(screen.getByRole('button', { name: /crear/i }));
    await waitFor(() => expect(mocked.createCategory).toHaveBeenCalledWith({ name: 'New' }));
  });

  it('confirms before deleting a category', async () => {
    const user = userEvent.setup();
    renderPage();
    await user.click(await screen.findByRole('button', { name: /eliminar pizza/i }));
    expect(screen.getByText(/¿seguro que querés eliminar/i)).toBeInTheDocument();
    await user.click(screen.getByRole('button', { name: /eliminar/i }));
    await waitFor(() => expect(mocked.deleteCategory).toHaveBeenCalledWith(1));
  });
});
```

---

## Implementation Steps

Ordered. Each step is atomic. Executor runs specs after each step.

### Step 1: Service

**Goal:** POST search + CRUD against `/katalog/<entity>` with typed response.
**Files:** `src/subscriptions/<slug>/services/<entity>Api.ts`

Run: `npm test -- src/subscriptions/<slug>/services/__tests__/<entity>Api.test.ts`

### Step 2: Hooks

**Goal:** `use<Entity>Form` (create/edit submit), `use<Entity>Delete` (confirm + call).
**Files:** `src/subscriptions/<slug>/hooks/use<Entity>Form.ts`, `use<Entity>Delete.ts`

Run: `npm test -- src/subscriptions/<slug>/hooks/__tests__/`

### Step 3: Columns + Modal

**Goal:** `<Entity>Columns.tsx` + `Create<Entity>Modal.tsx`.
**Files:** listed above.

Lint + typecheck: `npm run lint -- src/subscriptions/<slug>/pages/<Entity>/ && npm run typecheck`.

### Step 4: Index page

**Goal:** Wire `useRansackSearch` + `usePaginatedResource` + hooks. Follow the pattern in `context/pages.md` exactly.
**Files:** `src/subscriptions/<slug>/pages/<Entity>/index.tsx`

Run: `npm test -- src/subscriptions/<slug>/pages/<Entity>/__tests__/`

### Step 5: Route wiring

**Goal:** Add `<Route path="<entity>" element={<Entity />} />` to the subscription's route table.
**Files:** `src/subscriptions/<slug>/<Subscription>.tsx`, possibly `src/App.tsx`.

Verify by navigating manually in dev: `npm run dev`.

### Step 6: i18n keys

**Goal:** Add translation keys used in titles, buttons, messages.
**Files:** `src/i18n/<slug>/es.json` (and other languages if configured).

### Step 7: Coverage + lint + build

```bash
npm run quality:check
npm run build
```

Coverage must pass the per-file gate (75/65/75/70). Build must succeed with `CI=true`-like behavior.

---

## Checklist

- [ ] Specs written BEFORE implementing (RED before GREEN)
- [ ] All 4 states tested: Loading, Error, Success, Empty
- [ ] Queries use `getByRole` / `getByLabelText` over `getByTestId`
- [ ] `userEvent` not `fireEvent`
- [ ] No `any` anywhere
- [ ] No cross-subscription imports
- [ ] `useEffect` deps are exhaustive
- [ ] Refs used for mutable values read inside stable `useCallback`
- [ ] `key=<stable-id>` in every `.map()` — never `index`
- [ ] StrictMode-safe initial fetch via `didInitRef`
- [ ] Page 1 reset on search change
- [ ] `aria-label` on icon-only action buttons
- [ ] `ConfirmationDialog` before any destructive action
- [ ] Form inputs: `id`, `htmlFor`, `aria-invalid`, `aria-describedby`
- [ ] `useMemo`/`useCallback` justified (deps of another hook OR passed to memoized child)
- [ ] `AbortController` or `ignore` flag on async effects
- [ ] i18n keys for all user-facing strings
- [ ] `karta` not imported or rendered anywhere
- [ ] `npm run quality:check` passes locally
- [ ] `npm run build` passes locally
- [ ] Commit: Conventional Commits — `feat: <desc> - <TICKET>` etc.
- [ ] PR against `main`, body has Notion URL

---

## Open Questions

<Real blockers only. Omit if none.>

- <e.g., "Task mentions editing a 'price' field but the backend `Category` model in core-web-api doesn't expose `price` — add to backend first, or drop from AC?">
- <e.g., "Coverage comment in `.github/workflows/ci.yml` says 85% but `vitest.config.ts` enforces 75%. Align before shipping.">
````

---

## Step 7 — Confirm and present

After writing the plan file, output:

```
### Plan written

File:      plan-<TICKET>-<slug>.md
Branch:    <feature|bugfix|...>/<slug>-<TICKET>
Scope:     <subscription | global>
Entity:    <entity name>

Top 3 decisions:
  1. <e.g., "Recreate the CRUD pattern from Katalog::Categories: Service + 2 hooks + Index page + Create modal.">
  2. <e.g., "New route /<slug>/<entity> under the existing lazy subscription — no App.tsx refactor.">
  3. <e.g., "No new ENV vars; backend endpoint /katalog/<entity>/search already exists (verified via grep of core-web-api plan).">

Diagram generated: <yes + reason | no>

Open Questions (blockers for executor):
  - <list or "None">

Next step:
  Review the plan file. If OK, run:
    /customer-dev:execute plan-<TICKET>-<slug>.md
```

**Stop here.** Do not create branch, do not write code, do not modify anything. Wait for the user.

---

## Hard rules

- **Only plan.** No implementation. That's `customer-dev:execute`'s job.
- **Every new file anchored to a real analogue** with `path:line`. No analogue → cross-subscription analogue + explicit note.
- **Validate against the 13 context files.** Conflicts → "Open Questions."
- **Never invent a ticket ID.** Missing → `⚠️ NOT SPECIFIED`; warn that the commit/PR will need one.
- **Never skip acceptance criteria.** Each AC maps to at least one spec or a file.
- **Read in parallel** when searching multiple analogues.
- **If the scope is deprecated `karta`** → hard blocker in Open Questions.
- **If the task needs a backend endpoint that doesn't exist** → hard blocker; link to the backend task first.
- **Spec skeleton MUST cover the 4 states** (Loading, Error, Success, Empty) for every page test file.
