# Architecture — core-web-customer

Vite 7 SPA. React 18.3. TypeScript 5. React Router 7. Auth JWT con auto-refresh. Backend: Rails API con Ransack + Pagy.

## Directory map (the parts you'll actually touch)

```
src/
├── App.tsx                    # Router + lazy subscriptions + ThemeProvider + providers
├── main.tsx                   # createRoot, StrictMode, i18n bootstrap
├── config/
│   ├── api.ts                 # API_BASE_URL (Vite proxy /api/v1/customer in dev)
│   └── kodimProducts.ts       # Product catalog (sidebar, dashboard) — source of truth
├── contexts/
│   ├── AuthContext (in hooks/useAuth.tsx)
│   └── SubscriptionsContext (in hooks/useSubscriptions.tsx)
├── services/
│   ├── auth.ts                # apiFetch + token mgmt (JWT in localStorage, 401 auto-refresh)
│   └── api/
│       ├── client.ts          # searchWithRansack helper
│       ├── constants.ts       # DEFAULT_PAGE_SIZE = 25
│       ├── types.ts           # PaginationParams, PaginatedResponse, SearchFilters, ApiException
│       └── <domain>.api.ts    # onboarding, users, subscriptions, questions, search
├── hooks/                     # GLOBAL hooks (cross-subscription)
│   ├── useRansackSearch.ts
│   ├── usePaginatedResource.ts
│   ├── useAuth.tsx
│   ├── useSubscriptions.tsx
│   ├── useToast.ts
│   ├── useWhatsappEmbeddedSignup.ts
│   └── use-mobile.tsx
├── components/                # GLOBAL components
│   ├── ui/                    # shadcn/ui primitives (button, input, dialog, select, …)
│   ├── data-table/            # DataTable + header + pagination
│   ├── AppSidebar.tsx         # Sidebar shell
│   ├── ConfirmationDialog.tsx # Mandatory before destructive actions
│   ├── PageHeader.tsx         # Search input + action button (reused by every Index page)
│   ├── Modal.tsx
│   ├── Toast.tsx
│   ├── LoadingSkeleton.tsx
│   ├── ProtectedRoute.tsx
│   ├── OnboardingGuard.tsx
│   └── ErrorBoundary.tsx
├── layouts/
│   ├── DashboardLayout.tsx    # Authenticated shell (sidebar + outlet)
│   └── AuthLayout.tsx
├── pages/                     # Global pages (not tied to a subscription)
│   ├── Login.tsx, ForgotPassword.tsx, AuthCallback.tsx
│   ├── Dashboard.tsx, dashboard/
│   ├── Users/, Users.tsx
│   ├── Channels.tsx
│   ├── NotFound.tsx
│   └── onboarding/
├── subscriptions/             # Self-contained feature modules (the product)
│   ├── kathy/
│   ├── katalog/
│   ├── kalendar/
│   └── karta/                 # DEPRECATED — do not expose in UI
├── i18n/                      # react-i18next bootstrap + resources
├── types/                     # Shared types (ransack, domain)
└── utils/                     # Pure helpers (debounce, pagination mapping, iconMapper)
```

## Auth & session

- JWT in `localStorage` under keys: `kodim_access_token`, `kodim_user_id`, `kodim_user_data`, `kodim_tenant_id`.
- `apiFetch` wraps `fetch`: injects `Authorization: Bearer <token>`, on 401 serializes concurrent requests through `isRefreshing`/`pendingResolvers` and retries once after refresh.
- If refresh fails → clears session + redirects to `/login`.
- **Never** write token logic outside `src/services/auth.ts`. Use `apiFetch` everywhere.

## Routing

- `App.tsx` composes: `<Router>` → providers → `<Routes>`.
- Public routes: `/login`, `/forgot-password`, `/auth/callback`.
- Protected routes (wrapped in `ProtectedRoute` + `DashboardLayout`):
  - Global: `/dashboard`, `/users`, `/channels`, `/onboarding/*`.
  - Subscriptions: each subscription owns its prefix — `/katalog/*`, `/kathy/*`, `/kalendar/*`.
- Subscription modules are loaded via `React.lazy` — the entry file of each subscription is imported lazily to keep the initial bundle small.

## i18n

- `react-i18next` via `src/i18n/`. All user-visible copy goes through `t('key')` with resources split by namespace.
- Language detection via `i18next-browser-languagedetector`.
- Never hard-code user-facing strings in JSX.

## Theming

- `ThemeProvider` (`src/components/theme-provider.tsx`) + `ThemeToggle`. Dark mode class-based, driven by Tailwind's `darkMode: 'class'`.

## Build

- Vite 7, `vite.config.ts` sets up the `/api/v1/customer` proxy for local dev.
- `@/` alias → `src/` (set in Vite config + tsconfig paths + vitest config).
- Output goes to `dist/`. CI uploads it as an artifact.

## Rules that are easy to violate (read twice)

1. The Golden Rule: **a subscription imports only from itself or from global** (`@/components`, `@/hooks`, `@/services`, `@/utils`, `@/types`, `@/config`, `@/contexts`). See `subscriptions.md`.
2. **`src/subscriptions/karta/` exists but is deprecated.** Never expose it in UI. Don't import from it.
3. **All auth flows** go through `useAuth` + `apiFetch`. No bespoke token handling.
4. **Protected routes must be wrapped** in `ProtectedRoute` AND `DashboardLayout`.
5. Page-level data fetching uses `usePaginatedResource` + `useRansackSearch` — not bespoke `useEffect(fetch)` loops.
