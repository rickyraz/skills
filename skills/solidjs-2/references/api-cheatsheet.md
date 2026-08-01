# SolidJS 2.0 API cheatsheet

Detailed behavior reference. Read the section you need; this file is meant to
be consulted per-topic, not read start to finish.

**Contents:** [Imports & TypeScript](#imports-and-typescript-config) ·
[Reactivity core](#reactivity-core) · [Lifecycle](#lifecycle) ·
[Async data](#async-data) · [Actions & optimistic](#actions-and-optimistic-updates) ·
[Stores](#stores) · [Control flow](#control-flow) · [DOM](#dom) ·
[Context](#context)

## Imports and TypeScript config

The DOM runtime, store helpers, and JSX types moved:

```ts
// Web/DOM runtime
import { render, hydrate } from "@solidjs/web"; // was solid-js/web

// Stores - now exported from core, not a subpath
import { createStore, reconcile, snapshot, storePath } from "solid-js"; // was solid-js/store

// Hyperscript / tagged-template renderers, if used
import h from "@solidjs/h"; // was solid-js/h
import html from "@solidjs/html"; // was solid-js/html
import { createRenderer } from "@solidjs/universal"; // was solid-js/universal
```

`solid-js` no longer exports a JSX namespace or `jsx-runtime` - the core
package owns renderer-neutral component types, and renderer packages own JSX
types. Update `tsconfig.json` for web apps:

```json
{
  "compilerOptions": {
    "jsx": "preserve",
    "jsxImportSource": "@solidjs/web"
  }
}
```

Anywhere you imported `JSX` or `ComponentProps` from `solid-js`, import from
`@solidjs/web` instead. For renderer-neutral component prop types, use
`Element` from `solid-js` rather than `JSX.Element`:

```ts
import type { Component, Element } from "solid-js";
type Layout = Component<{ children?: Element }>;
```

## Reactivity core

### Signals

`createSignal(value)` is unchanged for the plain case. New: `createSignal(fn)`
creates a **writable derived signal** - it starts as whatever `fn()` computes,
but `setValue` can override it (think "writable memo"):

```ts
const [count, setCount] = createSignal(0);
const [label, setLabel] = createSignal(() => `Count: ${count()}`);
```

### Batching and flush

Every write is batched on a microtask automatically - there is no `batch()`
to call. A read immediately after a write in the same synchronous block still
returns the pre-write value:

```ts
const [a, setA] = createSignal(1);
setA(2);
a(); // still 1 here
flush(); // apply queued writes synchronously
a(); // now 2
```

Use `flush()` only when you need a synchronous "settled now" point - tests,
or interop with imperative DOM code (e.g. focusing an element right after a
state change). Don't sprinkle it into component logic defensively.

### `createEffect` - split into compute and apply

```ts
createEffect(
  computeFn, // tracked: read the things you depend on, return a value
  applyFn?, // untracked: receives (value, prevValue), may return cleanup
  options?
);
```

```ts
// Before (1.x): a single function did tracking and side effects together.
// After (2.0): tracking and side effects are two explicit functions.
createEffect(
  () => title(),
  (value) => {
    document.title = value;
  }
);
```

There's no `initialValue` argument anymore. `compute` receives `prev` (the
value it returned last time; `undefined` on the first run) as its argument -
give it a default parameter if you need one:

```ts
createEffect(
  (prev = 0) => count(),
  (value, prev) => console.log(`went from ${prev} to ${value}`)
);
```

Cleanup lives on the apply side, returned as a function:

```ts
createEffect(
  () => channel(),
  (name) => {
    const socket = connect(name);
    return () => socket.close();
  }
);
```

`EffectBundle` form for structured error handling - pass an object instead of
a bare `apply` function:

```ts
createEffect(() => riskyRead(), {
  effect: (value) => handleSuccess(value),
  error: (err) => handleFailure(err),
});
```

`createMemo`'s second argument is now `options`, not an initial value:

```ts
const doubled = createMemo(() => count() * 2); // no initialValue arg
const lazyOne = createMemo(() => expensive(), { lazy: true }); // deferred first run, auto-disposes at zero subscribers
```

### `on` is gone - split effects replace it

The compute half of a split effect already *is* the explicit dependency
declaration, so there's nothing left for `on` to do:

```ts
// Multiple explicit deps: just read them in compute.
createEffect(
  () => [a(), b()] as const,
  ([a, b]) => console.log(a, b)
);

// `defer` (skip the initial run) is now a plain option.
createEffect(() => count(), (v) => console.log(v), { defer: true });
```

### `createComputed` is gone - pick the right replacement

`createComputed` used to cover three different jobs; each has a direct 2.0
replacement, and they are not interchangeable:

| What the old code was doing              | Use instead                                    |
| ------------------------------------------ | ------------------------------------------------ |
| Pure derivation, no side effect            | `createMemo(() => ...)`                          |
| Side effect on change (logging, storage)   | split `createEffect(compute, apply)`             |
| Derived value that also needs a setter     | `createSignal(fn)`                                |

### Errors

`onError` / `catchError` are gone. Component-level error UI uses `Errored`;
programmatic handling inside an effect uses the `error` option on
`createEffect` (see `EffectBundle` above).

```tsx
<Errored fallback={(err, reset) => (
  <div>
    <p>{String(err)}</p>
    <button onClick={reset}>Try again</button>
  </div>
)}>
  <Widget />
</Errored>
```

`fallback` here is a **render callback**, not a component - it's invoked
directly with `(err, reset)` positional arguments by `Errored`, so it can't
be mounted as `<Fallback />`. Name it camelCase if you extract it, to signal
"callback" rather than "component". The same applies to the function-child
forms of `Show`, `For`, `Switch`/`Match`, and `Repeat`.

## Lifecycle

`onMount` is gone. `onSettled` is the closest replacement, and it can return
a cleanup function - covering the common "run once after the first
settle, tear down on dispose" pair that used to be `onMount` + `onCleanup`:

```ts
onSettled(() => {
  const observer = new ResizeObserver(measure);
  observer.observe(el);
  return () => observer.disconnect();
});
```

`onCleanup` still exists but is now a narrower, advanced primitive: reactive
cleanup tied to a computation's re-run, used when authoring custom reactive
primitives. For ordinary component/custom-hook setup-and-teardown, use
`onSettled` with a returned cleanup, not `onCleanup` directly. `onCleanup`
cannot be called from inside `onSettled` or `createTrackedEffect` - both of
those manage cleanup exclusively through their return value, and doing so
throws in dev (`CLEANUP_IN_FORBIDDEN_SCOPE`).

## Async data

There's no `createResource`. Any computation can return a Promise or async
iterable, and consumers just read the accessor as usual:

```ts
// Before: const [user] = createResource(userId, fetchUser);
const user = createMemo(() => fetchUser(userId()));
```

Wrap the part of the tree that reads a not-yet-ready value in `Loading`:

```tsx
<Loading fallback={<Spinner />}>
  <Profile user={user()} />
</Loading>
```

`Loading` covers **initial** readiness for its subtree. Once that subtree has
rendered once, further revalidation keeps the old content on screen instead
of tearing back down to the fallback - use `isPending` for "refreshing…" UI
on top of stable content:

```tsx
const refreshing = () => isPending(() => user());

<Loading fallback={<Spinner />}>
  <Show when={refreshing()}><RefreshBadge /></Show>
  <Profile user={user()} />
</Loading>;
```

`isPending(fn)` actually performs the read inside `fn` - place it under the
`Loading` boundary that owns that data, not off on its own reading only
upstream state (which can't tell you a *lower* subtree is pending):

```ts
isPending(id); // wrong if id itself is never async - always false
isPending(() => user()); // right - reads the async value directly
```

`Loading`'s `on` prop controls when the boundary re-shows its fallback during
revalidation instead of holding stale content - useful for route/key-level
transitions:

```tsx
<Loading on={id()} fallback={<Spinner />}>
  <Profile user={user()} />
</Loading>
```

Other async helpers:

- **`latest(fn)`** - peek at the in-flight value of a signal/computation
  during a transition, falling back to the stale value if the next one
  isn't ready yet. Useful for keeping a route param or input display in sync
  with what's actually loading.
- **`resolve(fn)`** - returns a Promise that resolves once `fn()` settles to
  a non-pending value. Can't be called from inside a reactive scope. Mainly
  for tests and imperative glue: `const value = await resolve(() => data());`.
- **`refresh(target)`** - explicitly invalidates/recomputes a derived source
  (a signal/store/projection created with a function form). Call it from
  event handlers, effects, or actions - not from pure computations. A bare
  `refresh()` that re-asks the same question is intentionally "quiet"
  (`isPending` stays `false`) unless paired with `affects()`.
- **`affects(target, key?)`** - declares that in-flight work will change
  `target`, so `target` (and anything derived from it) reads as pending from
  the declaration until the transaction settles or reverts, even though the
  actual change hasn't landed in the graph yet. `affects(store)` marks the
  whole store; `affects(record, "key")` marks one slot; `affects(accessor)`
  marks a signal/memo. Pair with `refresh()` when a reload should visibly
  read as pending rather than silently updating.

`Loading`/`Errored`/async reads compose with SSR: an uncaught pending read in
a render effect defers the root mount until it settles rather than throwing
(`ASYNC_OUTSIDE_LOADING_BOUNDARY`, a warning, not an error) - explicit
`Loading` boundaries are for when you want fallback UI or partial progressive
mount, not a requirement for correctness.

## Actions and optimistic updates

`action()` wraps a generator (or async generator) mutation. Inside it you can
do optimistic writes, `yield` async work, and call `refresh()`:

```ts
const [todos, setOptimisticTodos] = createOptimisticStore(
  () => api.listTodos(),
  []
);

const addTodo = action(function* (text: string) {
  setOptimisticTodos((list) => {
    list.push({ id: crypto.randomUUID(), text, done: false });
  });
  yield api.createTodo(text);
  refresh(todos);
});
```

Each `yield` is a network round trip - a bulk operation should call a bulk
endpoint and `yield` once, not loop `yield` per item.

- **`createOptimistic(value)`** - same surface as `createSignal`, but writes
  are optimistic: they show immediately and revert automatically when the
  surrounding transition completes (whether it commits or fails).
- **`createOptimisticStore(fnOrValue, seed, options?)`** - the store version.
  `fnOrValue` is typically a function returning the server-authoritative
  data; mutate it optimistically inside actions, then `refresh()` after the
  write lands. `options.key` defaults to `"id"` for reconciliation - only
  pass it when your data uses a different identity field, not to restate the
  default.
- **`refresh(target)`** - see above; the reconciliation step after a write.
- **`affects(target, key?)`** - see above; use when a mutation changes data
  that isn't directly reflected by an optimistic write (e.g. a server-set
  `updatedAt` timestamp), so the UI reads as pending on that slot too.

Design mutations as one derived source with layered concerns, not several
parallel stores you manually keep in sync - see `references/anti-patterns.md`
finding on "one store, not three."

## Stores

### Draft-first setters (mutation is the default now)

Store setters receive a mutable draft - the equivalent of 1.x's `produce`
wrapper is now just how setters behave, so nothing needs wrapping:

```ts
const [state, setState] = createStore({ user: { name: "" }, items: [] as string[] });

setState((draft) => {
  draft.user.name = "Alice";
  draft.items.push("first item");
});
```

A setter can also **return** a value for shallow replacement instead of
mutating: arrays are replaced index-by-index with length adjusted (so
surviving object references keep their identity), objects get a top-level
shallow diff. This is most useful for filtering:

```ts
setState((draft) => ({ ...draft, items: [] })); // full replace
setItems((list) => list.filter((item) => item.id !== targetId)); // remove
```

This return-path replacement is **not** keyed reconciliation - that only
applies to the *projection function* form (`createStore(fn, seed, { key })`,
`createOptimisticStore`, `createProjection`), where the function's return
value is diffed against the previous draft by `options.key`. Don't conflate
the two: a plain setter returning a filtered array works because of
positional index-replacement on same-reference survivors, not because of
keying.

If you want the old path-argument ergonomics, `storePath` is an opt-in
compatibility helper:

```ts
setState(storePath("user", "name", "Alice"));
```

### Derived stores: `createStore(fn, seed)`

```ts
const [todos] = createStore(() => api.listTodos(), []);
const [summary] = createStore(
  (draft) => {
    draft.total = todos().length;
  },
  { total: 0 }
);
```

### `createProjection(fn, seed)`

A derived store with reactive reconciliation - the general-purpose version of
the pattern `createStore(fn)`/`createOptimisticStore` use, for building your
own derived-with-reconciliation primitives.

### Plain values and prop helpers

```ts
const plain = snapshot(store); // was unwrap(store)
JSON.stringify(plain);

const merged = merge(defaults, overrides); // was mergeProps
const rest = omit(props, "class", "style"); // was splitProps(props, [...])
```

One behavioral difference in `merge`: `undefined` is a real value that
overrides, not a signal to "skip this key" - `merge({ a: 1 }, { a: undefined }).a`
is `undefined`.

### `deep(store)` and `reconcile(value, key)`

`deep(store)` makes a store's observation deep - by default a store only
tracks the properties actually read, `deep` opts a subtree into tracking all
nested changes. `reconcile(value, key)` is the diffing function used
internally by the projection-form primitives to update stores from new data
by identity; use it directly if you're writing your own reconciliation logic
against fetched data.

### `createMutable` is gone

Use `createStore` with draft setters instead - writes go through `setState`,
which is what lets them participate in batching and transitions. A raw
mutable proxy has no such hook.

## Control flow

### Lists: `Index` is gone, `For` covers both keying modes

```tsx
// keyed={false} is the direct Index replacement: item and index are
// both accessors, and identity is purely positional.
<For each={items()} keyed={false}>
  {(item, i) => <Row label={item()} position={i()} />}
</For>

// Default / keyed={true}: identity-keyed, item is the raw value.
<For each={items()}>
  {(item, i) => <Row label={item} position={i()} />}
</For>
```

Prefer a literal `keyed` value; a dynamic `keyed={cond()}` makes the callback
shape ambiguous since the two modes pass different argument types.

`Show`, `Switch`/`Match`, and `Repeat` also pass accessors into their
function children so reads stay safe regardless of when the child re-runs:

```tsx
<Show when={user()} fallback={<Login />}>
  {(u) => <Profile user={u()} />}
</Show>
```

### `dynamic(source)` factory (replaces `createDynamic`)

```tsx
import { Dynamic, dynamic } from "@solidjs/web";

// JSX wrapper - unchanged call site, now backed by dynamic() internally.
<Dynamic component={isEditing() ? Editor : Viewer} value={value()} />;

// Factory form - returns a stable component reference.
const ActiveView = dynamic(() => (isEditing() ? Editor : Viewer));
<ActiveView value={value()} />;
```

### `Reveal` (replaces `SuspenseList`)

Coordinates reveal timing across sibling `Loading` boundaries via an `order`
prop: `"sequential"` (default - reveal front to back), `"together"` (wait for
the whole group), or `"natural"` (the group participates as one slot in its
parent's ordering, then its children reveal independently once released). A
`collapsed` boolean, meaningful only under `order="sequential"`, keeps later
fallbacks hidden until their turn.

```tsx
<Reveal>
  <Loading fallback={<Skeleton />}><Header /></Loading>
  <Loading fallback={<Skeleton />}><Body /></Loading>
</Reveal>
```

## DOM

- **Attributes are attributes**, generally lowercase, closer to raw HTML -
  not silently mapped to DOM properties. Booleans are presence/absence
  (`muted={true}` adds the attribute, `muted={false}` removes it).
- **`attr:`, `bool:`, and `on:` namespaces are removed** - you don't need
  them. Keep using camelCase handlers (`onClick`) for Solid-managed events;
  for native listener options (e.g. `{ capture: true }`), attach via a ref
  callback instead of `on:`.
- **`use:` directives are removed.** Replace with `ref` directive factories,
  which compose via array:

  ```tsx
  <button ref={[autofocus, tooltip({ content: "Save" })]} />
  ```

  Prefer the two-phase shape for a directive factory: an owned setup phase
  (create primitives/subscriptions) and an unowned apply phase (the actual
  DOM write) returned as a function of the element:

  ```ts
  function tooltip(config: TooltipConfig) {
    let el: HTMLElement | undefined;
    createEffect(
      () => config.content,
      (content) => {
        if (el) el.title = content;
      }
    );
    return (nextEl: HTMLElement) => {
      el = nextEl;
    };
  }
  ```

- **`classList` is folded into `class`**, which accepts string, array, and
  object forms combined:

  ```tsx
  <div class={["card", { active: isActive(), disabled: isDisabled() }]} />
  ```

- **`/*@once*/` is removed** from the public JSX model - it is not a JSX
  form of `untrack`. Keep reactive values reactive in JSX; for a DOM
  element's initial-only state, use the platform's own default prop
  (`defaultValue` instead of freezing `value`); for a genuine one-time
  JavaScript read outside JSX, use `untrack(() => ...)` narrowly.
- **Delegated events are owned by render roots.** `render()`/`hydrate()`
  install and clean up delegated listeners on their own root container.
  `Portal` registers its outside-root mount point with the owning root so
  events still bubble through the logical Solid tree. If old code called
  `clearDelegatedEvents()`, drop it - disposing the render root replaces it.

## Context

The context itself is the provider - `Context.Provider` is gone:

```tsx
const Theme = createContext("light");
<Theme value="dark">{props.children}</Theme>;
```

A **default-less** context (`createContext<T>()`, no argument) is typed
`Context<T>`, not `Context<T | undefined>` - `useContext` returns `T`
directly and throws `ContextNotFoundError` at runtime if nothing provided
it. This removes the old `useX`-with-throw wrapper hook pattern entirely;
call `useContext` directly:

```ts
const TodosContext = createContext<TodosApi>();
// useContext(TodosContext) is TodosApi, throws if no <TodosContext> ancestor.
```

A context created **with** a default (`createContext(defaultValue)`) is
unchanged - `useContext` falls back to it outside any provider. Reserve the
default form for genuinely static, ownerless values (theme, locale, frozen
config); reactive payloads (signals, stores, `[state, actions]` tuples)
generally can't have a sensible default because they need an owner to be
constructed, so use the default-less throwing form for those.

Prefer plain props until drilling is genuinely deep or fans out to several
unrelated branches (roughly 3+ hops as a rule of thumb) - Context has real
cost (an extra indirection, a provider you must remember to mount) and
isn't the default anti-prop-drilling tool the way it is in React. Do not use
a module-level singleton as an alternative to Context for shared app state:
module scope has no disposal and, in an SSR app, is shared across concurrent
requests.
