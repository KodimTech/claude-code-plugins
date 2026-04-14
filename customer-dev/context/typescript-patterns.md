# TypeScript Patterns — React-specific idioms

The project is strict TS. No `any`. This file covers the React-specific patterns a Rails dev won't know from other TS codebases.

## Event handlers — pick the right type

```ts
const onChange = (e: React.ChangeEvent<HTMLInputElement>) => setName(e.target.value);
const onClick  = (e: React.MouseEvent<HTMLButtonElement>) => {/*…*/};
const onSubmit = (e: React.FormEvent<HTMLFormElement>) => { e.preventDefault(); };
const onKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {/*…*/};
const onFocus  = (e: React.FocusEvent<HTMLInputElement>) => {/*…*/};
```

Generic element type in the angle brackets matters: `e.currentTarget` is typed as that element. Prefer `currentTarget` over `target` — `target` can be any child of the event's listener.

## Refs

```ts
// DOM node ref
const inputRef = useRef<HTMLInputElement | null>(null);
// Mutable value ref
const didInitRef = useRef<boolean>(false);
// Ref to a function (rare, but valid)
const callbackRef = useRef<() => void>();
```

- Always include `| null` for DOM refs (`useRef` with no initial value returns `T | undefined`).
- For forwarded refs: `React.forwardRef<HTMLInputElement, Props>((props, ref) => …)`.

## Children

```ts
// Explicit children prop
interface CardProps {
  title: string;
  children: React.ReactNode;
}

// Convenience wrapper (adds `children?: React.ReactNode`)
interface CardProps extends React.PropsWithChildren<{ title: string }> {}

// Narrower: only strings
interface Props {
  children: string;
}

// Render prop
interface Props {
  children: (value: number) => React.ReactNode;
}
```

Prefer `React.ReactNode` for anything renderable. Avoid `JSX.Element` (too narrow — excludes strings, numbers, fragments).

## Component signature

```ts
// ✅ Preferred — inferred return type, explicit props type
interface Props {
  title: string;
  onCancel: () => void;
}

export const Categories = ({ title, onCancel }: Props) => {
  return <div>{title}</div>;
};

// ✅ Also fine
export const Categories: React.FC<Props> = ({ title, onCancel }) => <div>{title}</div>;
```

`React.FC` is controversial but OK — just know it implicitly adds `children`, which can be surprising. Most of this codebase uses the explicit-props form.

## Generics in custom hooks

When the hook operates on an entity type supplied by the caller:

```ts
export function usePaginatedResource<T>(
  fetchFn: (p: { filters?: SearchFilters; pagination?: PaginationParams }) => Promise<PaginatedResponse<T>>,
  options?: UsePaginatedOptions
): UsePaginatedApi<T> {
  const [data, setData] = useState<T[]>([]);
  // …
}

// Consumer
const { data } = usePaginatedResource<Category>(fetchCategories);
```

Rule: avoid `any` inside the hook body — type it with `T`. If the caller doesn't know T at usage time, your hook is poorly typed.

## Discriminated unions — great for forms / async state

```ts
type FetchState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string };

function render(state: FetchState<User>) {
  switch (state.status) {
    case 'idle':    return null;
    case 'loading': return <Spinner />;
    case 'success': return <UserCard user={state.data} />;     // TS knows `data` is User
    case 'error':   return <ErrorMsg msg={state.error} />;     // TS knows `error` is string
  }
}
```

This is stronger than three separate booleans (`isLoading`, `isError`, `data | null`). TS enforces that you check before accessing.

## `as const` — pin literal types

```ts
const FIELDS = ['name', 'email'] as const;  // readonly ['name', 'email']
type Field = (typeof FIELDS)[number];        // 'name' | 'email'

const CONFIG = { predicate: 'cont' } as const;  // { readonly predicate: 'cont' }
```

Useful for:
- Ransack predicate fields (`predicate: 'cont' as const`).
- Lookup objects that should be narrowed to their literal keys.
- Avoiding string widening (`'cont'` → `string`) in object literals.

## `satisfies` — type-check without widening

```ts
const routes = {
  dashboard: '/dashboard',
  login: '/login',
} satisfies Record<string, string>;

routes.dashboard;  // type: '/dashboard' (still the literal, not widened to string)
```

Use `satisfies` instead of `: Record<string, string>` when you want to preserve literal types.

## Narrowing `unknown` (preferred over `any`)

```ts
function parseError(err: unknown): string {
  if (err instanceof ApiException) return err.message;
  if (err instanceof Error) return err.message;
  if (typeof err === 'string') return err;
  return 'Unknown error';
}
```

Catch clauses default to `unknown` in modern TS — narrow before using.

## Props destructuring defaults

```ts
interface Props {
  debounceMs?: number;
  disabled?: boolean;
}

export function useRansackSearch({ debounceMs = 300, disabled = false }: Props) {
  // debounceMs: number  (default applied)
  // disabled: boolean
}
```

## Utility types you'll use

| Type | Use |
|---|---|
| `Partial<T>` | All keys optional. Good for `initialData` in forms. |
| `Pick<T, 'a' \| 'b'>` | Narrow a type to specific keys. |
| `Omit<T, 'a'>` | Drop keys. |
| `Required<T>` | All keys required. |
| `Readonly<T>` | Mark keys readonly. |
| `ReturnType<typeof fn>` | Derive a type from a function's return. |
| `Parameters<typeof fn>` | Derive a tuple of a function's params. |
| `NonNullable<T>` | Strip `null` / `undefined`. |

## Common missteps (caught in review)

- ❌ `interface Props { data: any[] }` → `data: Category[]` or generic.
- ❌ `as unknown as Foo` double-cast → sign of a wrong type somewhere upstream.
- ❌ `Function` type → use `(arg: X) => Y`.
- ❌ `object` type → use a proper shape or `Record<string, unknown>`.
- ❌ `// @ts-ignore` without a justification → use `// @ts-expect-error <reason>` so TS tells you when the ignore becomes unnecessary.
- ❌ Ambient global declarations (`declare global { … }`) for things that should be imports.

## Typing i18next

```ts
import { useTranslation } from 'react-i18next';

const { t } = useTranslation('katalog');
const title = t('categories.title');  // string
```

The project has typed resource keys via `i18next` module augmentation (check `src/i18n/`). When you add a translation, add the key type too.

## React Router 7 types

```ts
import { useParams, useSearchParams, useNavigate } from 'react-router-dom';

const { id } = useParams<{ id: string }>();   // id: string | undefined
const [params, setParams] = useSearchParams(); // URLSearchParams
const navigate = useNavigate();
navigate('/katalog/categories');               // returns void
```

`useParams` returns partial — narrow before using.
