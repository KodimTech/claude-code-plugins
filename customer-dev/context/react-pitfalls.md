# React Pitfalls — the traps a non-expert will hit

Read `react-mental-model.md` first. This file is the concrete catalogue of mistakes, each with a fix.

---

## 1. Stale closures in `useEffect`

**Smell**: your effect uses a value but doesn't list it in deps; the value inside the effect is "frozen."

```ts
const [user, setUser] = useState(initial);

useEffect(() => {
  if (user) saveToAnalytics(user);  // captures `user` at effect-run time
}, []);  // ❌ missing dep
```

**Fix**: list the dep, or use a ref if you genuinely need the latest value without re-running the effect.

Rule: **`exhaustive-deps` is always right.** If it's wrong, the hook design is wrong.

---

## 2. Missing / wrong deps in `useCallback` / `useMemo`

```ts
const fetch = useCallback(async () => {
  const r = await search({ ransackQuery });   // ⚠️ captures ransackQuery
  return r;
}, []);  // ❌
```

Either list the dep (`[ransackQuery]`) or read it from a ref you update each render:

```ts
const ransackQueryRef = useRef(ransackQuery);
ransackQueryRef.current = ransackQuery;

const fetch = useCallback(async () => {
  const r = await search({ ransackQuery: ransackQueryRef.current });
  return r;
}, []);  // ✅ stable identity + always-fresh read
```

This is the exact pattern used in `Categories/index.tsx`.

---

## 3. `key={index}` in mapped lists

```tsx
{items.map((item, i) => <Row key={i} item={item} />)}  // ❌
```

Fails on reorder, filter, insert. State inside `<Row>` attaches to the wrong item.

```tsx
{items.map((item) => <Row key={item.id} item={item} />)}  // ✅
```

If you truly have no id (e.g., a local list of form rows), generate one:

```ts
const rows = useMemo(() => initial.map((r) => ({ ...r, _key: crypto.randomUUID() })), [initial]);
```

---

## 4. Derived state antipattern

```ts
function Editor({ name }: { name: string }) {
  const [localName, setLocalName] = useState(name);  // ❌ state derived from prop
  // ...
}
```

Problem: if `name` changes from the parent, `localName` won't follow. The state goes stale silently.

**Fixes, in order of preference**:
1. **Compute during render**: if it's just a transformation, don't store it.
2. **Lift state up**: if the parent should own it, let the parent.
3. **Use `key` to reset child state**: `<Editor key={user.id} name={user.name} />` — when `user.id` changes, React unmounts and remounts the editor, resetting its state.
4. **Reset explicitly with `useEffect`** (last resort): track a previous value and reset when it changes.

---

## 5. Race conditions in async effects

```ts
useEffect(() => {
  fetchUser(id).then(setUser);
}, [id]);
```

If `id` changes quickly (fast typing, navigation), responses can arrive out of order and the wrong user "wins." Always guard:

```ts
useEffect(() => {
  const controller = new AbortController();
  fetchUser(id, { signal: controller.signal })
    .then(setUser)
    .catch((err) => { if (err.name !== 'AbortError') throw err; });
  return () => controller.abort();
}, [id]);
```

If your fetch API doesn't support `signal`, use an `ignore` flag:

```ts
useEffect(() => {
  let ignore = false;
  fetchUser(id).then((u) => { if (!ignore) setUser(u); });
  return () => { ignore = true; };
}, [id]);
```

---

## 6. Using `useEffect` for things that aren't effects

`useEffect` is for **synchronizing with external systems** (network, DOM APIs, subscriptions). Symptoms that you're misusing it:

| Misuse | Fix |
|---|---|
| Derive state from props/state | Compute during render or with `useMemo`. |
| Transform data for display | Compute during render. |
| Reset state when a prop changes | Use `key`, or lift state up. |
| Notify a parent | Call the parent's callback from the event handler that caused the change. |
| Run on button click | Just run it in the click handler — no effect needed. |

React's docs have the canonical list: https://react.dev/learn/you-might-not-need-an-effect.

---

## 7. State updates on unmounted components (React warning gone but bug remains)

React 18 removed the warning, but the bug isn't gone — you can still update state on an unmounted component if you don't clean up async work. The render output is thrown away, but the memory leak and the "ghost update" are real.

Always clean up:
- `AbortController` on fetch.
- `clearTimeout` / `clearInterval` on timers.
- `unsubscribe` on pub/sub.

---

## 8. Rendering a new object/array every render as a prop

```tsx
<Child config={{ retries: 3 }} />  // ⚠️ new object every render
```

If `<Child>` is `React.memo`-wrapped, it will re-render anyway because the prop identity changed. Memoize the prop:

```tsx
const config = useMemo(() => ({ retries: 3 }), []);
<Child config={config} />
```

Or, when the object is a constant, move it outside the component:

```tsx
const CONFIG = { retries: 3 };
function Parent() { return <Child config={CONFIG} />; }
```

---

## 9. Conditional hooks (rules-of-hooks violation)

```ts
if (user) {
  const [x, setX] = useState(0);  // ❌ — hooks must run in the same order every render
}
```

Hooks must be called in the same order on every render. No `if`, no `return` before a hook. If you need conditional logic, move it INSIDE the hook:

```ts
const [x, setX] = useState(0);
if (!user) return null;
```

---

## 10. `useState` with a function as initial value

```ts
const [user, setUser] = useState(fetchUserSync());   // ❌ runs on every render (throwaway)
const [user, setUser] = useState(() => fetchUserSync());  // ✅ runs once
```

`useState(value)` evaluates `value` every render. `useState(() => value)` evaluates only on mount.

---

## 11. Setting state inside render

```tsx
function Component() {
  const [x, setX] = useState(0);
  if (someCondition) setX(42);   // ❌ infinite loop
  return <div>{x}</div>;
}
```

Allowed only as a one-time correction (rare) — wrap in a conditional that holds a ref to track if already corrected. Usually this is a sign of derived state; fix the design instead.

---

## 12. `useReducer` + context pattern without memoization

The Provider re-renders the entire subtree every time state changes. If the context value is an unmemoized object, every consumer re-renders. If the reducer's state is a reference type that gets recreated, same. Memoize the value:

```tsx
const value = useMemo(() => ({ state, dispatch }), [state]);
```

---

## 13. Forgetting that `onClick={foo()}` calls immediately

```tsx
<button onClick={handleClick()}>...</button>    // ❌ calls on every render
<button onClick={handleClick}>...</button>       // ✅ passes the function
<button onClick={() => handleClick(id)}>...</button>  // ✅ for args
```

---

## 14. Accidentally passing `false` / `0` into JSX

```tsx
{items.length && <List items={items} />}  // ❌ renders the literal "0" when empty
{items.length > 0 && <List items={items} />}  // ✅
```

---

## 15. Over-using Context for everything

Context is great for auth, theme, i18n, feature flags — values that rarely change. Bad for "shared state that changes often" — you'll re-render every consumer. Use prop-drilling, or a proper state manager if the scale demands it.

---

## 16. `forwardRef` confusion

If your custom input needs to expose its DOM node (e.g., to focus programmatically), use `forwardRef`:

```tsx
const Input = React.forwardRef<HTMLInputElement, Props>((props, ref) => {
  return <input ref={ref} {...props} />;
});
```

Without `forwardRef`, refs passed to your component are ignored.

---

## 17. `StrictMode` double-invocation

React 18 dev StrictMode intentionally:
- Renders components twice.
- Runs every effect twice (mount → cleanup → remount).

Purpose: surface bugs where your component/effect doesn't handle being re-run cleanly. **If your feature breaks in StrictMode, it has a real bug.** Do not disable StrictMode to "fix" it.

This is why initial fetches use the `didInitRef.current` guard pattern — to avoid double-fetching in dev while keeping the benefit of StrictMode detection.

---

## 18. Forgetting error boundaries

A render-time error in a child component unmounts the entire tree unless an `ErrorBoundary` catches it. Wrap subscription roots (or at least the app root) in `ErrorBoundary` — it's already provided at `src/components/ErrorBoundary.tsx`.

---

## 19. `dangerouslySetInnerHTML` without sanitization

```tsx
<div dangerouslySetInnerHTML={{ __html: userContent }} />  // ⚠️ XSS
```

If you absolutely must, sanitize with DOMPurify first, and comment why. The plugin hooks will flag this.

---

## 20. `useLayoutEffect` SSR warnings

Using `useLayoutEffect` in code that might run server-side logs a warning and falls back to `useEffect`. This app is client-only (SPA), so `useLayoutEffect` is fine — but don't reach for it without a reason (measuring layout before paint is the main one).
