# Pages — CRUD patterns (Index, Create, Edit)

Every subscription CRUD follows the same shape. Learn it once, reuse everywhere. The reference implementation is `src/subscriptions/katalog/pages/Categories/index.tsx`.

## Index page anatomy

Responsibilities:
1. Wire `useRansackSearch` + `usePaginatedResource`.
2. Compose form + delete hooks.
3. Render `PageHeader` + `DataTable` + modals + confirmation dialog.
4. Guard initial fetch (StrictMode-safe).
5. Reset to page 1 when search changes.

### Skeleton

```tsx
import { useCallback, useEffect, useMemo, useRef, useState } from 'react';
import { PageHeader, ConfirmationDialog } from '@/components';
import { DataTable } from '@/components/data-table';
import { useRansackSearch } from '@/hooks/useRansackSearch';
import { usePaginatedResource } from '@/hooks/usePaginatedResource';
import { useToast } from '@/hooks/useToast';
import { DEFAULT_PAGE_SIZE } from '@/services/api';
import type { RansackSearchField } from '@/types/ransack';
import { mapArrayToPaginated } from '@/utils/pagination';
import { getCategoryColumns } from './CategoriesColumns';
import { CreateCategoryModal } from './CreateCategoryModal';
import { useCategoryForm } from '../../hooks/useCategoryForm';
import { useCategoryDelete } from '../../hooks/useCategoryDelete';
import { searchCategories, type Category } from '../../services/categoriesApi';

export const Categories: React.FC = () => {
  const toast = useToast();
  const [isActionDisabled, setIsActionDisabled] = useState(false);

  // 1. Search config (memoized — new identity would reset debounce every render)
  const SEARCH_CONFIG: RansackSearchField = useMemo(
    () => ({ fields: ['name'], predicate: 'cont' as const }),
    []
  );

  const { searchValue, setSearchValue, ransackQuery } = useRansackSearch({
    searchConfig: SEARCH_CONFIG,
    debounceMs: 300,
  });

  // 2. Refs over state for mutable params read inside fetch (avoids stale closures)
  const ransackQueryRef = useRef(ransackQuery);
  ransackQueryRef.current = ransackQuery;

  // 3. Fetch function — stable identity, reads ref at call time
  const fetchCategories = useCallback(
    async ({ filters, pagination }: { filters?: SearchFilters; pagination?: PaginationParams }) => {
      const result = await searchCategories({
        ransackQuery: { ...ransackQueryRef.current, ...filters },
      });
      return mapArrayToPaginated(result, pagination);
    },
    []
  );

  // 4. Pagination machinery
  const {
    data, loading, error, currentPage, pageSize, totalResults, totalPages,
    fetchData, setPage, refetch, hasFetchedRef,
  } = usePaginatedResource<Category>(fetchCategories, { pageSize: DEFAULT_PAGE_SIZE });

  // 5. Form + delete hooks
  const categoryForm = useCategoryForm({
    onSuccess: async () => {
      await refetch();
      toast.success(categoryForm.editing ? 'Categoría actualizada' : 'Categoría creada', ...);
    },
    onActionStateChange: setIsActionDisabled,
  });

  const categoryDelete = useCategoryDelete({
    onSuccess: async () => { await refetch(); toast.success('Categoría eliminada', ...); },
    onError: (msg) => toast.error('Error', msg),
    onActionStateChange: setIsActionDisabled,
  });

  // 6. Initial fetch (StrictMode-safe)
  const didInitRef = useRef(false);
  useEffect(() => {
    if (didInitRef.current) return;
    didInitRef.current = true;
    void fetchData();
  }, [fetchData]);

  // 7. Reset to page 1 on search change (skip first render)
  useEffect(() => {
    if (!hasFetchedRef.current) return;
    setPage(1);
    void fetchData(undefined, { page: 1 }, false);
  }, [ransackQuery, fetchData, hasFetchedRef, setPage]);

  const columns = useMemo(
    () => getCategoryColumns(
      (cat) => categoryForm.openEdit(cat),
      (cat) => categoryDelete.setTarget(cat),
      isActionDisabled,
    ),
    [categoryForm, categoryDelete, isActionDisabled]
  );

  return (
    <div>
      <PageHeader
        title="Categorías"
        searchValue={searchValue}
        onSearchChange={setSearchValue}
        searchLoading={loading}
        actionLabel="Nueva categoría"
        onAction={categoryForm.openCreate}
      />
      <DataTable
        data={data}
        columns={columns}
        loading={loading}
        error={error}
        emptyMessage="No hay categorías todavía."
        pageIndex={currentPage - 1}
        pageSize={pageSize}
        totalResults={totalResults}
        totalPages={totalPages}
        onPageChange={(idx) => setPage(idx + 1)}
      />
      <CreateCategoryModal {...categoryForm} />
      <ConfirmationDialog
        open={!!categoryDelete.target}
        title="Eliminar categoría"
        description={`¿Seguro que querés eliminar "${categoryDelete.target?.name}"?`}
        onConfirm={categoryDelete.confirm}
        onCancel={() => categoryDelete.setTarget(null)}
        destructive
      />
    </div>
  );
};
```

## Create/Edit — modal vs dedicated page

Two equally valid patterns in this codebase:

### Modal (preferred for simple entities)

`Create<Entity>Modal.tsx` holds the form. Opens from `categoryForm.openCreate()` or `openEdit(cat)`. Same component handles both modes via the `editing` prop.

- Less routing, less state. Good for 1-3 field forms.
- Close on success, refetch list, toast.
- Focus management: Radix `Dialog` handles it.

### Dedicated Create/Edit pages

Used when the form has multiple sections, file uploads, or complex validation (e.g., Products with steps).

```
src/subscriptions/katalog/pages/Products/
├── index.tsx                  # List
├── CreateProductModal.tsx     # or Create.tsx as a route
├── steps/                     # Multi-step form sections
```

- Navigate after success: `navigate('/katalog/products')`.
- Load initial data by ID in Edit before rendering the form (show `LoadingSkeleton` while loading).

## Rules — every CRUD page

- [ ] Uses `PageHeader` (search + primary action).
- [ ] Uses `DataTable` (never a custom `<table>`).
- [ ] Search is server-side via Ransack — never `array.filter(...)` in the component.
- [ ] Uses `useRef` + `didInitRef` for the initial fetch (StrictMode-safe).
- [ ] Resets to page 1 when `ransackQuery` changes.
- [ ] `searchLoading={loading}` wired on `PageHeader`.
- [ ] `size` + `minSize` set on every column.
- [ ] `ConfirmationDialog` before every destructive action.
- [ ] Action buttons have `aria-label`.
- [ ] `onActionStateChange` disables other actions during mutation (can't delete while creating).
- [ ] `pageIndex - 1` / `+ 1` conversion between TanStack and backend.
- [ ] Toast on every mutation (success AND error path).
- [ ] Error state renders a message, not a blank screen.
- [ ] Empty state with an `emptyMessage` that is helpful (not "No data").

## Routing inside a subscription

`App.tsx` lazy-loads the subscription root. The subscription's own `Katalog.tsx` (or equivalent) defines nested routes:

```tsx
<Routes>
  <Route index element={<KatalogDashboard />} />
  <Route path="categories" element={<Categories />} />
  <Route path="products" element={<Products />} />
  <Route path="products/new" element={<CreateProduct />} />
  <Route path="products/:id/edit" element={<EditProduct />} />
</Routes>
```

Keep the subscription root thin — route definitions + Suspense fallback.

## What NOT to do

- ❌ Don't fetch inside the Index page's render body. Always through the hook.
- ❌ Don't store the full entity list in Context just to avoid prop-drilling a list between Index and a modal. Pass props or pull from the same hook in both.
- ❌ Don't use `useState` to mirror URL params — use `useSearchParams` from React Router.
- ❌ Don't navigate inside a `useEffect` without a guard — you'll get redirect loops under StrictMode.
- ❌ Don't skip the confirmation dialog for "just this one" destructive action.
