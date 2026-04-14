# Custom Hooks

The project uses two canonical global hooks for data-fetching patterns: `useRansackSearch` (debounced search) + `usePaginatedResource` (list pagination). Subscription-specific hooks sit alongside pages.

## `useRansackSearch` — debounced server-side search

Location: `src/hooks/useRansackSearch.ts`.

```ts
const SEARCH_CONFIG: RansackSearchField = useMemo(
  () => ({ fields: ['name', 'email'], predicate: 'cont' as const }),
  []
);

const { searchValue, setSearchValue, ransackQuery, clearSearch } = useRansackSearch({
  searchConfig: SEARCH_CONFIG,
  debounceMs: 300,
});
```

Returns:
- `searchValue` — immediate (controlled input binding).
- `ransackQuery` — debounced, built as `{ q: { fieldA_or_fieldB_cont: value } }`.
- `clearSearch` — resets both.

Rules:
- **Always memoize the `searchConfig`** (`useMemo`) — otherwise you create a new config every render, re-triggering the debounce.
- Wire `searchLoading={loading}` on `PageHeader` so users see the spinner while the debounced fetch runs.
- Reset to page 1 when `ransackQuery` changes (the paginator needs to forget the old page).

Predicates available: `eq`, `not_eq`, `cont`, `not_cont`, `start`, `end`, `gt`, `lt`, `gteq`, `lteq`, `in`, `null`, `present`, `blank`.

## `usePaginatedResource` — list state + pagination

Location: `src/hooks/usePaginatedResource.ts`.

```ts
const fetchCategories = useCallback(
  async ({ filters, pagination }) => {
    const result = await searchCategories({ ransackQuery: ransackQueryRef.current });
    return mapArrayToPaginated(result, pagination);
  },
  []
);

const {
  data, loading, error, currentPage, pageSize, totalResults, totalPages,
  fetchData, setPage, setFilters, refetch, hasFetchedRef,
} = usePaginatedResource<Category>(fetchCategories, { pageSize: DEFAULT_PAGE_SIZE });
```

Key internals to understand:
- **Refs over state for "current" values**: `currentPageRef`, `currentFiltersRef`, `isFetchingRef`. Reason: avoids stale closures inside `fetchData` and prevents double-fetches in StrictMode.
- **`hasFetchedRef`** — exposed so the page can tell whether it's still in first-load (`loading && !hasFetchedRef.current` → full skeleton) vs. re-fetching (`loading && hasFetchedRef.current` → keep current rows, show overlay spinner).
- **Auto-refetch on page change**: `setPage(n)` internally fetches. You don't need to manually re-fetch.

Rules:
- Wrap the fetch function in `useCallback` with stable deps. If deps change every render, pagination will thrash.
- **Read mutable search params via refs**, not state, inside the fetch function — avoids stale closure bugs. See the `ransackQueryRef.current = ransackQuery` pattern in `Categories/index.tsx`.

## Initial fetch pattern (StrictMode-safe)

React 18 StrictMode mounts effects twice in dev. To prevent double-fetch:

```ts
const didInitRef = useRef(false);
useEffect(() => {
  if (didInitRef.current) return;
  didInitRef.current = true;
  void fetchData();
}, [fetchData]);
```

This is the idiomatic pattern in this codebase. Use it on every page that auto-fetches on mount.

## Page-reset-on-filter pattern

```ts
useEffect(() => {
  if (!hasFetchedRef.current) return;   // skip on initial render
  setPage(1);                            // reset page
  void fetchData(undefined, { page: 1 }, false);
}, [ransackQuery, fetchData, hasFetchedRef, setPage]);
```

When `ransackQuery` changes after the first render, reset to page 1 before fetching.

## Subscription-level hooks — CRUD glue

Location: `src/subscriptions/<slug>/hooks/use<Entity><Action>.ts`.

Three common shapes you'll see:

### Form hook (Create/Edit state + submit)

```ts
export const useCategoryForm = ({ onSuccess, onActionStateChange }: Opts) => {
  const [editing, setEditing] = useState<Category | null>(null);
  const [name, setName] = useState('');
  const [submitting, setSubmitting] = useState(false);

  const submit = useCallback(async () => {
    setSubmitting(true);
    onActionStateChange?.(true);
    try {
      editing
        ? await updateCategory(Number(editing.id), { name })
        : await createCategory({ name });
      await onSuccess?.();
      reset();
    } finally {
      setSubmitting(false);
      onActionStateChange?.(false);
    }
  }, [editing, name, onSuccess, onActionStateChange]);

  return { editing, setEditing, name, setName, submitting, submit, /* ... */ };
};
```

### Delete hook (confirmation + call + callback)

```ts
export const useCategoryDelete = ({ onSuccess, onError, onActionStateChange }: Opts) => {
  const [target, setTarget] = useState<Category | null>(null);
  const confirm = useCallback(async () => {
    if (!target) return;
    onActionStateChange?.(true);
    try {
      await deleteCategory(Number(target.id));
      await onSuccess?.();
    } catch (err) {
      onError?.(err instanceof ApiException ? err.message : 'Error');
    } finally {
      onActionStateChange?.(false);
      setTarget(null);
    }
  }, [target, onSuccess, onError, onActionStateChange]);
  return { target, setTarget, confirm };
};
```

### Stats / read-only hook

```ts
export const useProductStats = () => {
  const [data, setData] = useState<ProductStats | null>(null);
  // ... standard fetch-on-mount with abort controller ...
  return { data, loading, error, refetch };
};
```

## Rules for custom hooks

- **Exported name**: `use<Entity><Action>` (`useCategoryForm`, `useProductDelete`).
- **Stable function returns**: wrap in `useCallback` anything passed back to consumers (so it can be safely put in `useEffect` deps).
- **Return a stable object**: if the hook's return value is consumed in a memoized deps list, wrap it in `useMemo` to keep identity stable.
- **Errors**: surface via a callback (`onError`), not by re-throwing into render — render-time errors unmount the tree.
- **`useCallback` deps**: list every dependency (the `exhaustive-deps` lint rule is your friend). Never silence the rule.
- **Cleanup async**: if the hook does `await fetch(...)`, guard with `AbortController` so a late response doesn't set state after unmount.

## What NOT to do

- ❌ Don't build a `useCategories()` that also owns pagination state + search state + CRUD — that's three hooks in a trenchcoat. Split: data (`usePaginatedResource` generic) + form (`useCategoryForm`) + delete (`useCategoryDelete`).
- ❌ Don't use `useEffect` to derive state from props. Compute during render or with `useMemo`.
- ❌ Don't set state inside the render body. It's allowed for a one-time correction, but it's a code smell — usually means derived state.
- ❌ Don't return functions that aren't wrapped in `useCallback`. Consumers will have unstable deps, causing re-renders.
