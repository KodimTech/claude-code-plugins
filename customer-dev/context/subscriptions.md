# Subscriptions — the multi-tenant module rule

Each Kodim product is a **subscription**: a self-contained module under `src/subscriptions/<slug>/`. The customer panel is the shell; subscriptions are the features.

## Current subscriptions

| Slug | Product | Audience | Status |
|---|---|---|---|
| `kathy` | AI agent (FAQs + channels) | all | active |
| `katalog` | Product catalog | restaurants, shops | active |
| `kalendar` | Bookings / service calendar | clinics, service businesses | active |
| `karta` | Menu management (legacy) | — | **DEPRECATED** — folder may exist, must not appear in UI |

Product catalog source of truth: `src/config/kodimProducts.ts`. Sidebar + dashboard read from it. **Update this file** when adding/removing products. **Do not list `karta`.**

## Folder shape (per subscription)

```
src/subscriptions/<slug>/
├── pages/
│   ├── <Entity>/
│   │   ├── index.tsx                 # List (Index) page
│   │   ├── <Entity>Columns.tsx       # DataTable column defs (size + minSize)
│   │   ├── Create<Entity>Modal.tsx   # Create modal (or Create.tsx page)
│   │   └── ...
│   └── <Subscription>.tsx            # Subscription landing / dashboard (optional)
├── services/
│   └── <entity>Api.ts                # apiFetch wrappers, entity + form-data types
├── hooks/                            # Optional — for CRUD/form/delete hooks
│   ├── use<Entity>Form.ts
│   ├── use<Entity>Delete.ts
│   └── ...
└── components/                       # Optional — only if components are reused across
                                      # multiple pages of the same subscription
```

If a component is used by only one page, keep it in `pages/<Entity>/`. Only promote to `components/` inside the subscription when 2+ pages use it.

## The Golden Rule (non-negotiable)

> **A subscription NEVER imports from another subscription.**

❌ Forbidden:
```ts
// Inside src/subscriptions/kathy/…
import { Category } from '@/subscriptions/katalog/services/categoriesApi';
```

✅ Correct when two subscriptions need the same code — **promote it**:
```
src/hooks/useEntityDelete.ts         # shared hook (if the pattern is truly generic)
src/components/<Thing>.tsx           # shared UI
src/types/<thing>.ts                 # shared types
src/services/api/<domain>.api.ts     # shared API surface
```

Why: products should ship independently. Coupling two subscriptions locks their release cadence.

## Allowed imports per file location

| File lives in | May import from |
|---|---|
| `src/subscriptions/<slug>/**` | own subscription (`./`, `../`) + global (`@/components`, `@/hooks`, `@/services`, `@/utils`, `@/types`, `@/config`, `@/contexts`, `@/lib`, `@/layouts`) |
| `src/components/**` | `@/hooks`, `@/services`, `@/utils`, `@/types`, `@/lib` (NOT subscriptions) |
| `src/hooks/**` | `@/services`, `@/utils`, `@/types`, `@/lib` (NOT subscriptions, NOT components that aren't primitives) |
| `src/services/**` | `@/utils`, `@/types`, `@/config` (NOT subscriptions, NOT components, NOT hooks) |

## Promotion checklist — moving code from subscription to global

When you realize a piece of code is used across subscriptions:

1. **Confirm it's truly generic** — if it embeds domain terms (`"kalendar_resource"`, `"katalog_item"`) it's not generic, decompose first.
2. Move to the right global folder (see table above).
3. Rename to a domain-neutral name.
4. Update imports in all consumers.
5. Add tests for the global version (it now has more blast radius).
6. Verify: `npm run typecheck && npm run lint && npm run test:threshold`.

## Naming within a subscription

| Element | Convention | Example |
|---|---|---|
| Page component | PascalCase | `Categories` (exported from `Categories/index.tsx`) |
| Column defs file | `<Entity>Columns.tsx` | `ProductsColumns.tsx` |
| Modal component | `<Action><Entity>Modal.tsx` | `CreateCategoryModal.tsx` |
| Hook | `use<Entity><Action>.ts` | `useProductForm.ts`, `useCategoryDelete.ts` |
| Service file | `<entity>Api.ts` (plural or singular — follow the existing neighbours) | `categoriesApi.ts`, `productsApi.ts` |
| Entity type | PascalCase | `Category`, `Product` |
| Form data type | `<Entity>FormData` | `CategoryFormData` |
| URL path | kebab-case plural | `/katalog/categories`, `/katalog/products` |

## Kalendar — backend endpoint quirks

Kalendar is the subscription with the strongest UI↔backend terminology mismatch. See `services.md` for the mapping — the short version: frontend calls them "servicios", backend calls them `/kalendar/resources`. Always verify the endpoint name when touching Kalendar services.
