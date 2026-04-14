# React Mental Model — how React actually thinks

If you're coming from Rails (or any MVC), React's mental model is the single biggest source of confusion. This file exists to bridge the gap.

## The one-liner

> **A React component is a pure function that, given props + state, returns a description of the UI. React is responsible for turning that description into DOM updates. You describe WHAT; React figures out HOW.**

Contrast with Rails: in ERB you imperatively build HTML, and the browser renders it once per request. In React, the component re-runs **every time state or props change**, and React diffs the new output against the previous one.

## The rendering cycle

```
1. render phase    — React calls your component function. You return JSX (a tree of elements).
2. reconciliation  — React compares the new tree to the previous tree (Virtual DOM diff).
3. commit phase    — React applies the minimal DOM changes needed.
4. effects         — AFTER commit, React runs `useEffect` callbacks (in the order they were defined).
5. cleanup         — BEFORE the next effect run (or on unmount), React runs the previous effect's cleanup function.
```

The component function runs in the render phase. Effects run in the effect phase. **You can't perform DOM side-effects directly in the render phase** — put them in `useEffect`.

## Props & state — what triggers a re-render

A component re-renders when:
1. Its **state** changes (`setState` / `setX` from `useState`).
2. Its **parent** re-renders (which passes "new" props — even if the values are the same, the reference may differ).
3. A **context value** it consumes changes.
4. A **`useReducer` dispatch** fires.

### The crucial detail: React re-runs the WHOLE function body

Every `useState`, `useMemo`, `useRef`, local variable — all re-executed. The only thing preserved between renders is:
- The value stored in `useState`/`useReducer`/`useRef`.
- The memoized result of `useMemo`/`useCallback` (if deps are equal).

Everything else is recomputed from scratch.

## Closures — the source of "stale state" bugs

Every render, your component function creates a new closure over the current `state` and `props`. An effect defined in that render captures those specific values.

```ts
const [count, setCount] = useState(0);

useEffect(() => {
  const id = setInterval(() => {
    console.log(count);  // ⚠️ captures `count` at the time the effect ran
  }, 1000);
  return () => clearInterval(id);
}, []);  // ❌ empty deps — effect only runs once, captures count=0 forever
```

The interval keeps logging `0`, not the current value, because the effect never re-runs and the closure is frozen. Fix:

```ts
useEffect(() => {
  const id = setInterval(() => console.log(count), 1000);
  return () => clearInterval(id);
}, [count]);  // ✅ re-creates the interval whenever count changes
```

Or use a ref that you update imperatively:

```ts
const countRef = useRef(count);
countRef.current = count;

useEffect(() => {
  const id = setInterval(() => console.log(countRef.current), 1000);
  return () => clearInterval(id);
}, []);  // ✅ effect is once; ref always has the latest value
```

**This is not optional knowledge.** Almost every "my component has stale data" bug is a closure problem.

## `useState` batching & async nature

`setX(value)` does NOT update state immediately. It **schedules** a re-render with the new value.

```ts
const [count, setCount] = useState(0);

const increment = () => {
  setCount(count + 1);
  setCount(count + 1);
  setCount(count + 1);
  // count is still 0 here. After this handler returns, React re-renders ONCE with count=1.
};
```

React 18 batches state updates **even inside promises / timeouts** (React 17 only batched inside event handlers).

To update based on the previous value, use the functional form:

```ts
setCount((prev) => prev + 1);
setCount((prev) => prev + 1);
setCount((prev) => prev + 1);
// After the handler returns, count is 3.
```

## `useEffect` is NOT `componentDidMount`

It's easy to think `useEffect(…, [])` is "runs once on mount." Mostly true — **but in StrictMode dev, it runs twice**, and the cleanup fires between the two runs. This is intentional: React wants to surface effects that don't clean up properly.

Treat every effect as "synchronize with an external system and clean up when done." If you can't write a clean cleanup, the effect is probably doing too much.

## Why `useEffect` fires AFTER the DOM update

Effects are deliberately deferred so they don't block painting. If you need to measure layout before paint (rare), use `useLayoutEffect`. Otherwise, stick with `useEffect`.

## `useRef` — persistent mutable box

`useRef(initial)` returns `{ current: initial }`. Writing to `ref.current` does **not** trigger a re-render. Refs persist across renders.

Use cases:
- Holding a DOM node (`<div ref={divRef} />`).
- Holding a mutable value read inside an effect (to avoid stale closures).
- Holding a `didInit` / `hasFetched` flag (StrictMode guard).
- Holding an `AbortController` for an ongoing fetch.

Do NOT use refs for data that affects rendering — state is for that.

## `useMemo` / `useCallback` — when do they help?

**Default: don't use them.** Most computations are fast; most props passed down are fine with a fresh identity each render.

Use them when:
1. The value is a **dependency of another hook**. New identity every render makes the dependent hook re-run (possibly forever).
2. The value is passed to a `React.memo`-wrapped child that does expensive rendering. Stable identity prevents re-render.
3. The computation itself is measurably expensive (confirmed with React DevTools Profiler).

**Measure before optimizing.** Cargo-cult memoization adds code weight without speeding anything up — and sometimes slows things down (memoization has its own cost).

## Lists & keys

When rendering a list, React uses `key` to match items across renders. Without stable keys, React falls back to index-based matching, which causes state to attach to the wrong item when the list reorders.

```tsx
// ❌ index as key — state attaches to the wrong row after sort/filter
items.map((item, i) => <Row key={i} item={item} />)

// ✅ stable unique id
items.map((item) => <Row key={item.id} item={item} />)
```

Why it matters: if `<Row>` has internal state (e.g., a toggle), index-keyed rows will "swap" their internal state when the array reorders. Users see their open row close, or edits jump to a different row.

## Context — a subtle re-render trap

`useContext(MyContext)` re-renders the consumer whenever the context **value reference** changes. If you do this:

```tsx
<MyContext.Provider value={{ user, setUser }}>
```

You pass a new object every render. Every consumer re-renders every time. Fix by memoizing the value:

```tsx
const value = useMemo(() => ({ user, setUser }), [user]);
return <MyContext.Provider value={value}>...</MyContext.Provider>;
```

Or split into two contexts (stable vs frequently-changing).

## The ONE thing to remember

> When something behaves oddly in React, **97% of the time it's a closure problem or a missing dependency**. Re-read the effect. Check what values it captures. Check the deps array.
