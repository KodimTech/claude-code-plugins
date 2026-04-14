# Rails → React glossary

If you work daily in core-web-api (Rails), this file maps concepts you already know to their React/TS equivalents. The analogies are approximate — they give you a mental hook, not perfect isomorphisms.

## Request lifecycle

| Rails | React |
|---|---|
| `routes.rb` → `ApplicationController` → action → view template | `<Routes>` in `App.tsx` → `ProtectedRoute` → page component → JSX return |
| `before_action :authenticate_user!` | `<ProtectedRoute>` wrapper + `useAuth()` guards |
| `before_action :set_resource` | `useEffect` that fetches on mount + the `didInitRef` guard |
| Per-request short-lived state | Per-component state (`useState`) — but the component can live across many interactions, so cleanup matters |

React components are not "per request." A component mounts, lives across many state changes, and unmounts. Effects (`useEffect`) are the closest thing to lifecycle callbacks.

## The MVC split

| Rails | React |
|---|---|
| Model (AR) | Entity type + service function (`Category`, `searchCategories()`). No in-memory ORM. |
| Controller | The page component's top-level code (decides what to render, handles events) |
| View (`.erb`) | JSX returned from the component |
| Partial | Component (reusable piece of JSX) |
| Helper | Hook (shared logic callable from many components) OR pure util function |
| Concern / mixin | Custom hook (for logic) OR HOC (for JSX wrapping — rare in this codebase) |
| `ApplicationController` ancestry | Context (`AuthContext`, `SubscriptionsContext`) for cross-cutting state |

## Data layer

| Rails | React |
|---|---|
| `Category.all` / `Category.where` | `searchCategories(params)` returning typed `Category[]` |
| `Category.find(id)` | `getCategory(id)` service function |
| `Category.create!` | `createCategory(data)` POST wrapper |
| ActiveRecord association | Backend relationship; frontend receives it as nested JSON (e.g., `item.item_categories`) |
| `accepts_nested_attributes_for` | Frontend sends `<relation>_attributes: [{ id, ..., _destroy: true }]` (see `services.md`) |
| `ransack` on the backend | `useRansackSearch` builds the query client-side; backend stays the same |
| `pagy` pagination | `usePaginatedResource` + `DEFAULT_PAGE_SIZE = 25` + Pagy headers parsed by `searchWithRansack` |
| Scopes (`Category.active`) | Predicates in the Ransack query (`{ q: { discarded_at_null: true } }`) |
| `current_user` / `acts_as_tenant` | `useAuth()` returns the current user; tenancy is backend-scoped — frontend just sends JWT + hits `/api/v1/customer/*` |

## Async work

| Rails | React |
|---|---|
| `perform_later` (Solid Queue) | No frontend equivalent — the frontend fires the HTTP call, the backend queues. Use `toast.success` after the call returns 202/accepted. |
| `ActionCable` / websockets | Not used in this app. If ever needed: `useEffect(() => { const ws = new WebSocket(…); return () => ws.close(); }, [])`. |

## State & persistence

| Rails | React |
|---|---|
| Instance variable (`@category`) per request | Local component state (`useState`) — but persists across renders until unmount |
| `session[:key]` | `localStorage` (used by `src/services/auth.ts` for JWT) OR React state lifted to the nearest common ancestor |
| `flash[:notice]` | `toast.success(title, body)` via `useToast` |
| `flash.now[:alert]` | Inline error state (`<span role="alert">`) or `toast.error` |

## Forms

| Rails | React |
|---|---|
| `form_with model: @category` | Controlled form component with local `formData` state |
| `form_with` strong params (`category_params`) | Service wrapper: `JSON.stringify({ katalog_category: formData })` |
| Server-side validation errors → `render :new` with errors | Catch `ApiException`, extract `.errors`, set `errors` state, re-render form |
| Custom validator in AR | Client-side `validate()` before submit (UX sugar) — server is still the authority |
| `simple_form` / `form_for` | shadcn/Radix primitives + manual wiring of `htmlFor` / `aria-*` |

## Error handling

| Rails | React |
|---|---|
| `rescue_from ActiveRecord::RecordInvalid` | `catch` around the service call; check `err instanceof ApiException` |
| `rescue_from StandardError` at the base controller | `ErrorBoundary` (global + per-subscription) to catch render errors |
| `Rails.logger.error` | `console.error` — with context |

## Request/response details

| Rails | React |
|---|---|
| `Authorization: Bearer …` | Injected automatically by `apiFetch` — don't build headers manually |
| `params[:q][:name_cont]` | Built by `useRansackSearch` and passed in the body as `{ q: { name_cont: … } }` |
| Response JSON from Blueprinter | Typed on the frontend: `(await response.json()) as Category[]` |
| HTTP status | `response.status` — `!response.ok` → throw `ApiException` |

## Testing

| Rails (RSpec) | React (Vitest + RTL) |
|---|---|
| `describe '#method'` | `describe('Component', () => { ... })` |
| `context 'when …'` | nested `describe` |
| `let(:user) { create(:user) }` | `beforeEach(() => mocked.fetchUser.mockResolvedValue({...}))` |
| `expect(…).to eq(…)` | `expect(…).toBe(…)` / `.toEqual(…)` |
| `instance_double(User)` | `vi.mocked(module).fn.mockResolvedValue(shape)` |
| Request spec | Render the page + mock the service layer + assert with `screen.getByRole` |
| `undercover` diff coverage | `vitest --coverage` + the `perFile` gate in `vitest.config.ts` |
| `doppler run -- bin/rspec` | `npm test` (no env-var wrapper needed — Vite `.env` is enough) |

## Styling / view logic

| Rails | React |
|---|---|
| Helper methods rendering HTML | Small presentational component |
| ERB `if/else` in views | Conditional JSX (`{condition && <…/>}` or `{condition ? <A/> : <B/>}`) |
| `content_tag(:div, 'hi', class: 'foo')` | `<div className="foo">hi</div>` |
| Asset pipeline / Sprockets | Vite bundler — assets imported directly (`import logo from './logo.svg'`) |
| Tailwind classes (if used in Rails) | Tailwind classes (identical syntax) |
| View partial locals | Component props |

## Security

| Rails | React |
|---|---|
| CSRF token | Not applicable for JSON API + JWT in `Authorization` header |
| `html_safe` / `raw()` | `dangerouslySetInnerHTML` — same risk, same "don't unless sanitized" rule |
| Strong params whitelist | Done on the backend; frontend just sends the wrapped payload |
| `request.xhr?` checks | Frontend is always "xhr" (SPA) — backend should treat it uniformly |

## Deployment

| Rails | React |
|---|---|
| Kamal + Doppler + `config/deploy.yml` | CI (GitHub Actions) builds `dist/`, uploads artifact — deploy infra is separate |
| `bin/ci` (rspec + rubocop + brakeman + signoff) | `npm run quality:check && npm run build` (lint + format + typecheck + test:threshold + build) |
| ENV vars in Doppler → `deploy.yml` → `.kamal/secrets` | `.env` + Vite's `import.meta.env.VITE_*` prefix. **No Doppler.** If adding a new env var, document it in the PR and update `.env.example`. |

## When analogies break down

- **Re-rendering**: Rails views render once per request. React components re-run many times during their lifetime. This is the single biggest mental shift. See `react-mental-model.md`.
- **Server state vs UI state**: Rails keeps almost everything server-side; the view is a snapshot. React has both (server state fetched via services + UI state in components). Keep the distinction clear — UI state shouldn't duplicate server state.
- **Callbacks vs events**: Rails `after_create` is a backend hook. React's "after something" is: (a) call the next thing in the event handler, or (b) `useEffect` with the right dep array. There's no global `after_create` equivalent — the component that owns the mutation is responsible for the follow-up.
- **Single-threaded vs multi-requested**: Rails handles many concurrent requests; each has its own memory space. React runs in one browser tab, one event loop. Race conditions between effects are frequent — see `react-pitfalls.md` #5.
