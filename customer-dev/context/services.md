# Service Layer — API client, Ransack, nested attributes

The service layer encapsulates all HTTP. Pages and hooks call services, never `fetch` directly.

## `apiFetch` — the only HTTP entry point

Location: `src/services/auth.ts`.

Behavior:
- Injects `Authorization: Bearer <access_token>` from `localStorage`.
- On `401`, triggers refresh flow, serializes concurrent requests, retries once. If refresh fails → logs out + redirects to `/login`.
- Returns the raw `Response` — callers decide how to parse.

```ts
import { apiFetch, API_BASE_URL, ApiException } from '@/services/api';

const response = await apiFetch(`${API_BASE_URL}/katalog/categories`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ katalog_category: { name } }),
});

if (!response.ok) {
  throw await ApiException.fromResponse(response);
}

const data = (await response.json()) as Category;
```

Rules:
- **Never** use raw `fetch()` for authenticated endpoints. Use `apiFetch`.
- **Always** `response.ok` check + `ApiException.fromResponse` on failure — don't silently return `null`.
- **Always** wrap POST/PATCH bodies in the Rails resource key: `{ katalog_category: {...} }`, `{ kalendar_resource: {...} }`. Flat payloads are a common bug.

## Service file shape

Location: `src/subscriptions/<slug>/services/<entity>Api.ts`.

```ts
import { apiFetch, API_BASE_URL, ApiException } from '@/services/api';

// Types live next to the service
export type Category = {
  id: string | number;
  name: string;
};

export type CategoryFormData = {
  name: string;
};

export async function searchCategories(params?: {
  sort?: string;
  ransackQuery?: Record<string, unknown>;
}): Promise<Category[]> { /* … */ }

export async function createCategory(data: CategoryFormData): Promise<Category> { /* … */ }
export async function updateCategory(id: number, data: CategoryFormData): Promise<Category> { /* … */ }
export async function deleteCategory(id: number): Promise<void> { /* … */ }
```

Rules:
- **No JSX, no React imports** in service files. They are pure functions.
- **Types in the same file** — `Entity` + `EntityFormData`. Import from `service.ts` consumers.
- **Return typed values**, never `any`. Cast the parsed JSON once at the boundary: `(await response.json()) as Category[]`.
- **Error surface**: throw `ApiException`. Callers catch it at the UI boundary.

## Ransack — server-side search

All listing endpoints that support filtering accept a Ransack query wrapped in `q`.

```ts
// Request shape
POST /katalog/categories/search
{ "q": { "name_cont": "pizza", "s": "name asc" } }

// Built by useRansackSearch
ransackQuery = { q: { name_or_description_cont: 'pizza' } }
```

`searchWithRansack` helper (in `src/services/api/client.ts`) wraps the POST + Pagy header parsing. Use it when the endpoint follows the pattern. When the endpoint deviates (e.g., no Pagy, returns a flat array) → call `apiFetch` directly, like `searchCategories` does.

Predicates available: `eq`, `not_eq`, `cont`, `not_cont`, `start`, `end`, `gt`, `lt`, `gteq`, `lteq`, `in`, `null`, `present`, `blank`. See `types/ransack.ts`.

## Pagination — Pagy

Backend returns pagination in response headers. `searchWithRansack` parses them into `PaginatedResponse<T>`:

```ts
{
  data: T[],
  pagination: { page, limit, total, totalPages }
}
```

For endpoints that return flat arrays (not paginated), use `mapArrayToPaginated(result, pagination)` from `@/utils/pagination` to fake the shape — `Categories/index.tsx` does this today.

## Nested attributes (Rails `accepts_nested_attributes_for`)

Backend returns join-table objects with a join `id`:

```json
{
  "id": 1,
  "name": "Item",
  "item_categories": [
    { "id": 123, "category": { "id": 2 } }
  ]
}
```

On update, send `_attributes` arrays with the join `id`:

```json
{
  "karta_item": {
    "item_categories_attributes": [
      { "id": 123, "category_id": 2 },           // update
      { "category_id": 3 },                         // create
      { "id": 124, "category_id": 4, "_destroy": true }  // delete
    ]
  }
}
```

Rules:
- **Read** (backend → form state): map `item.item_categories?.map(ic => ({ id: ic.id, category_id: ic.category.id }))`.
- **Write** (form state → payload):
  - Create: `{ category_id }` (no `id`).
  - Update: `{ id, category_id }`.
  - Delete: `{ id, category_id, _destroy: true }`.
- If the backend returns only `categories` without the join table → **demand that the backend returns the join** (`item_categories`, `item_ingredients`). Without it, updates are lossy.

## Kalendar — endpoint aliasing

UI terminology ≠ backend resource names. Always check:

| UI concept | Backend endpoint | Wrapper key |
|---|---|---|
| Tipos de Servicio | `/kalendar/resource_types` | `kalendar_resource_type` |
| Servicios | `/kalendar/resources` | `kalendar_resource` |
| Reservas | `/kalendar/bookings` | `kalendar_booking` |

```ts
// ❌ WRONG — using the UI name
apiFetch(`${API_BASE_URL}/kalendar/services`, ...);
body: JSON.stringify({ kalendar_service: formData });

// ✅ CORRECT — backend name
apiFetch(`${API_BASE_URL}/kalendar/resources`, ...);
body: JSON.stringify({ kalendar_resource: formData });
```

Function names and file names stay in UI terminology (`servicesApi.ts`, `searchServices()`) — but the URL path and wrapper key must match the backend.

## Errors — `ApiException`

```ts
try {
  await createCategory(data);
  toast.success('Categoría creada', `"${data.name}" creada.`);
} catch (err) {
  if (err instanceof ApiException) {
    toast.error('Error', err.message);
  } else {
    console.error(err);
    toast.error('Error', 'Algo salió mal. Intentá de nuevo.');
  }
}
```

- `ApiException.fromResponse(response)` reads the body and builds a friendly message.
- Surface user-facing text via `toast`. Never show a raw error payload.
- Log unknown errors with `console.error` for debugging.
