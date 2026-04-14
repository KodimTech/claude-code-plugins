# Performance — when memoization actually helps (and when it doesn't)

**Default position**: don't memoize. Modern React + Vite is fast. Memoization has a cost (closure allocation, comparison work, cognitive load). Reach for it only when a specific problem is measured.

## Measure first — the React DevTools Profiler

1. Open React DevTools → Profiler tab.
2. Hit "Record" → perform the interaction → stop.
3. Look at the flame chart:
   - **Width = time spent.** Wide bars = slow renders.
   - **Color gradient** shows which components re-rendered.
   - Click a component → see why it rendered ("Hooks changed", "Props changed: x, y").

If you can't reproduce a slow render in the Profiler, **don't optimize**. You'll make the code worse without speeding anything up.

## When `useMemo` actually helps

### 1. Expensive synchronous compute

```ts
const sorted = useMemo(
  () => data.slice().sort(expensiveCompare),
  [data]
);
```

"Expensive" = measurable. Sorting 10 items is not expensive. Filtering 100k items is. If you're unsure, don't memoize.

### 2. Referential stability for another hook's deps

```ts
const filters = useMemo(() => ({ status: 'active' }), []);

useEffect(() => {
  fetch(filters);
}, [filters]);   // ✅ stable reference, effect runs once
```

Without `useMemo`, `filters` is a new object every render, `useEffect` re-runs every render, you hit an infinite fetch loop.

### 3. Referential stability for `React.memo` children

```ts
const columns = useMemo(() => getColumns(onEdit, onDelete), [onEdit, onDelete]);
<DataTable columns={columns} />   // DataTable is React.memo'd; stable columns → skip re-render
```

## When `useCallback` actually helps

Same reasoning as `useMemo`, but for function identity.

```ts
const handleClick = useCallback(() => {
  doThing(id);
}, [id]);

<MemoChild onClick={handleClick} />   // stable function → MemoChild skips re-render
```

Without a memoized child or a hook deps list consuming the function, `useCallback` is waste — you create a closure and a deps-comparison cost every render, and gain nothing.

## When `React.memo` actually helps

`React.memo(Component)` skips re-render if props are shallow-equal to the previous render.

Use it when:
1. The component renders an expensive subtree (hundreds of nodes, complex layout).
2. Its parent re-renders often AND the component's props don't change often.

Don't use it when:
- The component renders a small amount of DOM. The diff is cheap; the memo comparison is close to wash.
- Props include objects/functions that are recreated every render (memo is useless without memoized props).
- It has context consumers that change frequently (memo doesn't stop context-driven re-renders).

## Lists — when to virtualize

The DataTable paginates (default 25 rows) → fine, no virtualization needed.

If you ever need to render 500+ rows at once (unlikely in this app), consider:
- `@tanstack/react-virtual` (matches the existing TanStack Table ecosystem).
- `react-window` (lighter, simpler).

Signs you need virtualization: scroll is visibly janky, initial render takes > 200ms, DOM has > 10k nodes.

## Lazy-loading routes

Already in use — subscriptions are `React.lazy`-imported in `App.tsx`. When adding a new subscription or a large page:

```tsx
const NewPage = React.lazy(() => import('./NewPage'));

<Suspense fallback={<PageLoader />}>
  <NewPage />
</Suspense>
```

Target: each subscription should be < ~150 KB gzipped. If a subscription grows past that, code-split its sub-routes.

## Bundle analysis

```bash
npm run build
# Then inspect dist/ — Vite prints a size summary
```

For a detailed analysis, install `rollup-plugin-visualizer` (not currently in the project) — propose as a dev-only dep in a PR if bundle growth becomes a concern.

## Images & assets

- Use SVG for icons via `lucide-react`. Don't bundle raster logos if SVG versions exist.
- Compress raster assets before committing (TinyPNG, Squoosh).
- Never import `node_modules` assets directly — Vite handles it, but check the `dist/` output.

## Re-render root causes — a triage flow

When a component re-renders too often:

1. Open the Profiler. Is it actually re-rendering? (Your intuition is often wrong.)
2. Click the component in the flame chart. What changed?
   - **"Props changed: foo"** → trace `foo` upstream. Is the parent creating a new object every render? `useMemo` the prop there.
   - **"Hooks changed"** → a context value or a `useState`/`useReducer` changed. If context, is the Provider value memoized?
   - **"Parent rendered"** → the parent re-rendered and this component isn't `React.memo`'d. Decide if it's worth memoizing.
3. If none of the above, check for `key` instability — a dynamic key causes unmount/remount (destroys state + re-runs all effects).

## Suspense & concurrent features — not needed here (yet)

React 18 Suspense for data is unstable without a framework (Next.js, Remix, Tanstack Router). This project uses it only for `React.lazy` component splits. Don't reach for `useTransition` / `useDeferredValue` without a measured problem — the debounced Ransack search already solves the typical "input-stutter" scenario.

## What NOT to do

- ❌ Wrap every function in `useCallback` "just in case." Adds code weight, no benefit.
- ❌ Memoize primitive values (`useMemo(() => 42, [])`). Primitives are already stable by value.
- ❌ `React.memo` a component with a `children` prop (children are usually new elements each render — memo always misses).
- ❌ Install `reselect` or a state-management lib for perf without proving the need.
- ❌ Pre-fetch all subscription modules on login "to make navigation snappy" — defeats lazy-loading and bloats the initial bundle. Prefetch on hover instead, if you must.
