# Testing — Vitest + React Testing Library

Stack: Vitest 3, @testing-library/react 16, @testing-library/user-event 14, jsdom.

## Coverage — what CI actually enforces

`vitest.config.ts` gates:

```
perFile: true
lines:       75
statements:  75
functions:   65
branches:    70
```

The CI comment says "85% threshold" — **this is misleading**; the real gate is 75/65/75/70 per-file. If CI docs and config disagree, flag it in the plan's Open Questions.

### Internal targets by file type (guidance, not per-file enforced)

| Type | Target |
|---|---|
| Services (`<entity>Api.ts`) | ≥ 80% |
| Hooks (`use<Entity>*.ts`) | ≥ 75% |
| Components | ≥ 70% |
| Pages | ≥ 65% |

## Exclusions (already in `vitest.config.ts`)

Coverage excludes a growing list of files (shadcn primitives, sidebar, some services). **Do not add new exclusions** without flagging — the exclusion list is a backlog, not a license. If your new file fails the threshold, write more tests.

## File layout

```
src/
├── subscriptions/katalog/pages/Categories/
│   ├── index.tsx
│   └── __tests__/
│       └── Categories.test.tsx
├── subscriptions/katalog/services/
│   ├── categoriesApi.ts
│   └── __tests__/
│       └── categoriesApi.test.ts
└── hooks/
    ├── useRansackSearch.ts
    └── __tests__/
        └── useRansackSearch.test.ts
```

Tests colocated in `__tests__/` next to the file under test. Name: `<File>.test.(ts|tsx)`.

## AAA structure

```ts
it('creates a category and refetches the list', async () => {
  // Arrange
  const user = userEvent.setup();
  vi.mocked(createCategory).mockResolvedValue({ id: 1, name: 'Pizza' });
  render(<Categories />);

  // Act
  await user.click(screen.getByRole('button', { name: /nueva categoría/i }));
  await user.type(screen.getByLabelText(/nombre/i), 'Pizza');
  await user.click(screen.getByRole('button', { name: /crear/i }));

  // Assert
  await waitFor(() => {
    expect(createCategory).toHaveBeenCalledWith({ name: 'Pizza' });
  });
  expect(await screen.findByText(/categoría creada/i)).toBeInTheDocument();
});
```

## Query priority (critical)

Prefer in this order:
1. `getByRole(name)` — closest to how users (and screen readers) perceive the UI.
2. `getByLabelText` — for form inputs.
3. `getByPlaceholderText` — weaker; prefer label when possible.
4. `getByText` — for non-interactive content.
5. `getByTestId` — **last resort**. If you need it, ask: is the role missing because of missing a11y?

Use `findBy*` for async appearance. Use `queryBy*` when asserting absence (returns null instead of throwing).

## `userEvent` over `fireEvent`

```ts
// ✅ userEvent — realistic
const user = userEvent.setup();
await user.type(input, 'hello');
await user.click(button);

// ❌ fireEvent — fires synthetic events without keyboard/focus semantics
fireEvent.change(input, { target: { value: 'hello' } });
```

Only use `fireEvent` for edge cases `userEvent` can't model (drag events, custom events).

## Mocking

### Mock the API at the module boundary

```ts
vi.mock('@/subscriptions/katalog/services/categoriesApi', () => ({
  searchCategories: vi.fn(),
  createCategory: vi.fn(),
  updateCategory: vi.fn(),
  deleteCategory: vi.fn(),
}));

import * as api from '@/subscriptions/katalog/services/categoriesApi';
const mocked = vi.mocked(api);

beforeEach(() => {
  mocked.searchCategories.mockResolvedValue([{ id: 1, name: 'Pizza' }]);
});
```

### Mock `apiFetch` for service tests

```ts
vi.mock('@/services/api', async () => ({
  ...(await vi.importActual('@/services/api')),
  apiFetch: vi.fn(),
  API_BASE_URL: '/api/v1/customer',
}));
```

Then set response shape per test:

```ts
vi.mocked(apiFetch).mockResolvedValue(
  new Response(JSON.stringify([{ id: 1, name: 'X' }]), { status: 200, headers: { 'Content-Type': 'application/json' } })
);
```

### Mock React Router

```ts
import { MemoryRouter } from 'react-router-dom';

render(
  <MemoryRouter initialEntries={['/katalog/categories']}>
    <Categories />
  </MemoryRouter>
);
```

For hooks that use `useNavigate`:

```ts
const navigate = vi.fn();
vi.mock('react-router-dom', async () => ({
  ...(await vi.importActual('react-router-dom')),
  useNavigate: () => navigate,
}));
```

## The 4 states — every component/page must test all 4

| State | What to test |
|---|---|
| Loading | Skeleton/spinner shown, `aria-busy="true"`, primary action disabled. |
| Error | Error message rendered, retry affordance visible, no stale data. |
| Success | Happy-path content renders. Interaction produces correct side-effect. |
| Empty | `emptyMessage` shown. Create-action still available. |

A test file without all 4 is incomplete.

## Async patterns

```ts
// ✅ waitFor (polls until assertion passes)
await waitFor(() => expect(screen.getByText(/cargado/i)).toBeInTheDocument());

// ✅ findBy (wraps waitFor + getBy)
const row = await screen.findByText(/pizza/i);

// ❌ setTimeout / manual wait
await new Promise((r) => setTimeout(r, 100));  // brittle and slow
```

## Custom hooks

Use `renderHook`:

```ts
import { renderHook, act } from '@testing-library/react';

const { result } = renderHook(() => useRansackSearch({ searchConfig: { fields: ['name'], predicate: 'cont' } }));

act(() => result.current.setSearchValue('pizza'));

await waitFor(() => {
  expect(result.current.ransackQuery).toEqual({ q: { name_cont: 'pizza' } });
});
```

Wrap in a provider if the hook uses context.

## Common mistakes

- ❌ **`act()` warnings** — almost always mean you're asserting before a state update settled. Use `await waitFor` / `findBy` / `await user.click`.
- ❌ **Not cleaning up `vi.mock`** — reset with `afterEach(() => vi.clearAllMocks())` or `mockReset()` per test.
- ❌ **Testing implementation details** — asserting component internal state via `screen` selectors that match implementation (e.g., classnames) rather than user-facing behavior.
- ❌ **`data-testid` everywhere** — usually means the component lacks proper roles. Fix the component.
- ❌ **Asserting snapshot of entire DOM** — brittle. Assert specific, user-observable facts instead.

## Test file template

```ts
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { MemoryRouter } from 'react-router-dom';
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { Categories } from '../index';

vi.mock('@/subscriptions/katalog/services/categoriesApi', () => ({
  searchCategories: vi.fn(),
  createCategory: vi.fn(),
  updateCategory: vi.fn(),
  deleteCategory: vi.fn(),
}));

import * as api from '@/subscriptions/katalog/services/categoriesApi';
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

  it('renders loading, then the list', async () => { /* … */ });
  it('shows an error state when fetch fails', async () => { /* … */ });
  it('shows the empty state when the list is empty', async () => { /* … */ });
  it('opens the create modal and creates a category', async () => { /* … */ });
  it('confirms before deleting a category', async () => { /* … */ });
});
```
