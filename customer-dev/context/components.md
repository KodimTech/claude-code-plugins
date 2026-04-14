# Components — Forms, Tables, shadcn/Radix, a11y

## Global components (reuse first)

Located in `src/components/`:

| Component | Purpose | When to use |
|---|---|---|
| `PageHeader` | Title + search input + primary action | Every Index page |
| `DataTable` | TanStack Table + pagination + empty/error states | Every list view |
| `ConfirmationDialog` | Destructive action confirmation | Before delete/discard |
| `Modal` | Controlled dialog wrapper (Radix) | Create/Edit flows without routing |
| `Toast` + `useToast` | Success/error notifications | After every mutation |
| `LoadingSkeleton` | Skeleton for loading states | Initial load when data is empty |
| `ErrorBoundary` | Catch render errors | Around subscription roots |
| `ProtectedRoute` | Auth gate | Around authenticated routes in `App.tsx` |
| `OnboardingGuard` | Blocks users mid-onboarding | After `ProtectedRoute` on most routes |
| shadcn `ui/*` | Primitives (Button, Input, Select, Dialog, …) | Form inputs, buttons, selects |

**Rule**: don't re-invent. If a primitive exists in `components/ui/`, use it.

## Form component pattern

Location: `src/subscriptions/<slug>/pages/<Entity>/<Entity>Form.tsx` (or inside a `Create<Entity>Modal.tsx`).

### Props contract

```ts
interface Props {
  initialData?: Partial<Entity>;
  onSubmit: (data: FormData) => Promise<void>;
  onCancel: () => void;
  isEdit?: boolean;
}
```

### State

```ts
const [formData, setFormData] = useState<FormData>({ ... });
const [errors, setErrors] = useState<Partial<Record<keyof FormData, string>>>({});
const [submitting, setSubmitting] = useState(false);
```

### Submit

```ts
const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault();
  if (!validate()) return;
  setSubmitting(true);
  try {
    await onSubmit(formData);
  } finally {
    setSubmitting(false);
  }
};
```

### Accessibility — mandatory checklist

For every input:
- [ ] Unique `id` (use `useId()` if rendered multiple times).
- [ ] `<label htmlFor={id}>` with visible label text.
- [ ] `aria-invalid={!!errors.field}` — announces invalid state.
- [ ] `aria-describedby={errors.field ? \`${id}-error\` : undefined}` — links error message.
- [ ] `required` on mandatory fields (plus a visible `*` in the label).
- [ ] Error message rendered with `id={\`${id}-error\`}` and `role="alert"` so screen readers announce it.

```tsx
const nameId = useId();

<div>
  <label htmlFor={nameId}>Nombre *</label>
  <input
    id={nameId}
    required
    value={formData.name}
    onChange={(e) => setFormData({ ...formData, name: e.target.value })}
    aria-invalid={!!errors.name}
    aria-describedby={errors.name ? `${nameId}-error` : undefined}
  />
  {errors.name && <span id={`${nameId}-error`} role="alert">{errors.name}</span>}
</div>
```

### Buttons

```tsx
<button type="submit" disabled={submitting}>
  {submitting ? 'Guardando…' : isEdit ? 'Actualizar' : 'Crear'}
</button>
<button type="button" onClick={onCancel}>Cancelar</button>
```

- Submit: `type="submit"`.
- Cancel: `type="button"` (otherwise it triggers submit).
- Disabled during `submitting`.
- Text swap on submit state.

### Controlled inputs — always

Every `<input>`, `<textarea>`, `<select>` **must** have `value` + `onChange`. No uncontrolled `defaultValue` unless you're using an explicit `ref` (rare, and documented when you do).

### Validation

- Validate inside `validate()` before submit.
- Return a boolean + populate `errors`.
- Clear the error for a field as soon as the user edits it (call `setErrors(prev => ({ ...prev, [field]: undefined }))` in `onChange`).

## Table component pattern

Location: `src/subscriptions/<slug>/pages/<Entity>/<Entity>Columns.tsx` + `index.tsx`.

### Columns file

```tsx
import { ColumnDef } from '@tanstack/react-table';
import { DataTableColumnHeader } from '@/components/data-table';

export const getCategoryColumns = (
  onEdit: (cat: Category) => void,
  onDelete: (cat: Category) => void,
  loading: boolean,
): ColumnDef<Category>[] => [
  {
    accessorKey: 'name',
    header: ({ column }) => <DataTableColumnHeader column={column} title="Nombre" />,
    size: 250,
    minSize: 200,
  },
  {
    id: 'actions',
    header: 'Acciones',
    cell: ({ row }) => (
      <div className="flex gap-2">
        <button aria-label={`Editar ${row.original.name}`} onClick={() => onEdit(row.original)} disabled={loading}>…</button>
        <button aria-label={`Eliminar ${row.original.name}`} onClick={() => onDelete(row.original)} disabled={loading}>…</button>
      </div>
    ),
    size: 150,
  },
];
```

### Rules

- Every column has `size` + `minSize`. Otherwise filtering causes layout shift.
- Action buttons have `aria-label` including the row value.
- Never render destructive actions without `ConfirmationDialog`.
- `pageIndex` is 0-based (TanStack). `currentPage` from `usePaginatedResource` is 1-based. Sync them:
  ```ts
  pageIndex={currentPage - 1}
  onPageChange={(idx) => setPage(idx + 1)}
  ```

### Index page skeleton

```tsx
<div>
  <PageHeader
    title="Categorías"
    searchValue={searchValue}
    onSearchChange={setSearchValue}
    searchLoading={loading}
    actionLabel="Nueva categoría"
    onAction={() => categoryForm.openCreate()}
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
    confirmLabel="Eliminar"
    onConfirm={categoryDelete.confirm}
    onCancel={() => categoryDelete.setTarget(null)}
    destructive
  />
</div>
```

## a11y — global rules

- **Semantic HTML first**: `<button>` over `<div onClick>`. `<nav>`, `<main>`, `<section>`, `<form>`.
- **One `<h1>` per page.** Sub-sections use `<h2>`, `<h3>`.
- **Focus management**: when a modal opens, focus the first interactive element. Radix handles this for `Dialog`.
- **Keyboard**: all interactive elements usable with Tab + Enter/Space. Never rely on mouse-only affordances.
- **Color**: don't convey meaning by color alone. Errors need a label/icon + color.
- **Loading states** wired with `aria-busy={true}` on the region, plus `role="status"` for screen-reader announcements.
- **`eslint-plugin-jsx-a11y`** is enabled. If it lints, fix the code — don't disable.

## shadcn/Radix

- Primitives in `src/components/ui/` are generated by shadcn. Don't edit them casually — regenerate with the shadcn CLI or carefully patch with a commit note.
- Radix already handles ARIA. Don't pile ARIA attributes on top of a Radix primitive (usually redundant, sometimes incorrect).
- Tailwind classes compose. Use `cn()` from `@/lib/utils` to merge them cleanly.

## Visual consistency

- Tailwind only. No inline `style={…}` unless it's a dynamic length/color that can't be expressed as a class.
- Theme: dark mode is class-based (`dark:*` variants in tailwind). Don't query `window.matchMedia` directly — use the `ThemeProvider`.
- Icons: `lucide-react`. Don't mix icon libraries.
