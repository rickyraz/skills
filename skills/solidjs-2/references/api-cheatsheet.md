# SolidJS 2.0 RC API cheatsheet

Snapshot baseline: **`solid-js@2.0.0-rc.1`**, `solidjs/solid` branch `next`,
verified 2026-08-21.

This is a condensed agent reference, not a replacement for current upstream
source.

## Imports and package layout

```ts
import {
  createSignal,
  createMemo,
  createEffect,
  createRenderEffect,
  createTrackedEffect,
  createRoot,
  createStore,
  createProjection,
  createOptimistic,
  createOptimisticStore,
  action,
  affects,
  refresh,
  isPending,
  latest,
  resolve,
  deep,
  snapshot,
  reconcile,
  merge,
  omit,
  storePath,
  untrack,
  flush,
  onCleanup,
  onSettled,
  createContext,
  useContext,
  For,
  Show,
  Switch,
  Match,
  Repeat,
  Loading,
  Errored,
  Reveal,
  children,
  lazy,
  DEV
} from "solid-js";

import {
  render,
  hydrate,
  Portal,
  dynamic,
  Dynamic
} from "@solidjs/web";
```

For web projects:

```json
{
  "compilerOptions": {
    "jsxImportSource": "@solidjs/web"
  }
}
```

Renderer packages own JSX types. Old entry points such as `solid-js/web` and
`solid-js/store` are not the 2.0 package layout.

## Signals and memos

### Plain signal

```ts
const [count, setCount] = createSignal(0);

count();
setCount(1);
setCount(c => c + 1);
```

Writes are queued.

```ts
setCount(1);
count(); // previous committed value

flush();
count(); // 1
```

### Readonly derived value

```ts
const doubled = createMemo(() => count() * 2);
```

### Writable derived value

Function-form `createSignal` is a writable memo-like primitive.

```ts
const [selected, setSelected] = createSignal(() => props.initial);
```

Useful signal/memo options include:

```ts
createSignal(0, {
  equals: (a, b) => a === b,
  ownedWrite: true,
  unobserved: () => cleanupExternalThing()
});

createMemo(expensive, {
  lazy: true,
  equals: false
});
```

`ownedWrite` is advanced. Do not add it just to silence
`REACTIVE_WRITE_IN_OWNED_SCOPE`.

### `loadingValue`

A memo can declare a committed provisional first value:

```ts
const feed = createMemo(
  () => fetchFeed(id()),
  {
    loadingValue: {
      provisional: true,
      items: placeholderItems
    }
  }
);
```

During the first flight:

- readers get `loadingValue`;
- they do not suspend to `Loading`;
- `isPending(feed)` stays false.

Use structural `Loading` instead when the placeholder is not honestly a value
of the same domain/type.

## Ownership and tracking

Low-level helpers expose the distinction:

```ts
getOwner();    // lifecycle/disposal owner
getObserver(); // active tracking observer, or null
```

A component body can be owned while a top-level read is untracked.

```tsx
function Bad(props: { count: number }) {
  const n = props.count; // dev warning
  return <span>{n}</span>;
}

function Good(props: { count: number }) {
  return <span>{props.count}</span>;
}
```

Use `untrack` only when a snapshot is intentional.

```ts
const initial = untrack(() => props.count);
```

## Effects

### `createEffect(compute, apply)`

Two phases:

```ts
createEffect(
  () => count(),
  (value, prev) => {
    console.log(value, prev);
    return () => cleanup();
  }
);
```

- compute is tracked;
- apply is untracked and side-effecting;
- apply is writable;
- apply may return a cleanup function.

One-argument `createEffect(compute)` is invalid in dev and emits
`MISSING_EFFECT_FN`.

### Error bundle

```ts
createEffect(
  () => fetchData(id()),
  {
    effect: data => {
      renderData(data);
    },
    error: (err, cleanup) => {
      console.error(err);
      cleanup();
    }
  }
);
```

The `error` arm handles compute/upstream errors. Throws from imperative effect
code escalate normally rather than being fed back into the same bundle.

### Store data in effects

Bad:

```ts
createEffect(
  () => store.user,
  user => {
    // store proxy properties are read in untracked apply phase
    send(user.name, user.age);
  }
);
```

Good:

```ts
createEffect(
  () => ({
    name: store.user.name,
    age: store.user.age
  }),
  user => send(user.name, user.age)
);
```

Deep observation:

```ts
createEffect(
  () => deep(store),
  plainSnapshot => persist(plainSnapshot)
);
```

Non-tracking plain snapshot:

```ts
const plain = snapshot(store);
```

### `createRenderEffect`

Same split idea, but render-phase/synchronous. Prefer normal `createEffect`
for application side effects unless render-phase semantics are required.

### `createTrackedEffect`

Special single-callback tracked leaf effect.

It is not the default effect form. Its callback is a restricted leaf owner:

- no nested reactive primitive creation;
- no `onCleanup`;
- return cleanup directly;
- a pending async read is not a normal suspendable use case.

## Batching and `flush`

`batch()` is removed. Updates batch by microtask.

```ts
setOpen(true);
flush();
dialog.focus();
```

`flush(fn)` is also supported for an imperative synchronous drain scope.

Do not use it as ordinary control flow.

Inside a normal effect apply callback, `flush()` is a no-op in dev and emits
`FLUSH_IN_EFFECT_CALLBACK`.

Inside `createTrackedEffect` / owner-backed `onSettled` while the queue is
running, re-entrant `flush()` throws; that throw currently has no stable
diagnostic code in the RC source.

## Lifecycle

### `onSettled`

Canonical component-level setup + teardown:

```ts
onSettled(() => {
  const observer = new ResizeObserver(measure);
  observer.observe(el);

  return () => observer.disconnect();
});
```

An owner-backed `onSettled` is implemented through a restricted leaf tracked
effect. Return cleanup; do not call `onCleanup` inside it.

An unowned/out-of-band `onSettled` can schedule work, but if its callback
returns cleanup there is no owner to attach it to and dev throws
`SETTLED_CLEANUP_UNOWNED`.

### `onCleanup`

Low-level lifecycle registration, mainly useful for custom primitive/library
internals.

```ts
const owner = getOwner();

runWithOwner(owner, () => {
  onCleanup(() => resource.dispose());
});
```

Dev behavior:

- no owner -> `NO_OWNER_CLEANUP` warning;
- restricted leaf scope -> `CLEANUP_IN_FORBIDDEN_SCOPE` error.

## Async data

Async belongs to normal computations.

```ts
const user = createMemo(() => fetchUser(id()));
```

### `Loading`

```tsx
<Loading fallback={<ProfileSkeleton />}>
  <Profile user={user()} />
</Loading>
```

`Loading` is a branch-readiness boundary.

Initial unresolved reads can show fallback. Once content has rendered,
ordinary revalidation keeps stale content visible.

Use `on` to re-arm fallback for a key change:

```tsx
<Loading on={id()} fallback={<ProfileSkeleton />}>
  <Profile user={user()} />
</Loading>
```

Nested boundaries are useful for progressive readiness.

### No `Loading` boundary

During the initial browser `render()` / `hydrate()` enforcement window, an
uncaught async read is legal: the root mount can be deferred until pending work
settles. Dev reports `ASYNC_OUTSIDE_LOADING_BOUNDARY` as an FYI warning.

Use a boundary when you want explicit fallback or partial progressive mount.

### `isPending`

```tsx
<Loading fallback={<Spinner />}>
  <Show when={isPending(() => user())}>
    Updating…
  </Show>
  <UserDetails user={user()} />
</Loading>
```

`isPending(fn)` performs the read in `fn`.

This is meaningful:

```ts
isPending(() => user());
```

This cannot infer a lower subtree's pending async merely because `id` caused it:

```ts
isPending(id);
```

### `latest`

Reads the in-flight view during a transition when available.

```ts
const visibleId = () => latest(id);
```

### `resolve`

Imperative/test helper that waits until an expression has a settled value.

```ts
const userValue = await resolve(() => user());
```

Do not call `resolve` from a reactive computation.

### `refresh`

```ts
refresh(user);
```

A bare refresh is a quiet re-ask of the same question. The old value remains
valid while the fresh answer arrives, so `isPending` need not turn true.

### `affects`

Declare data that in-flight work will change:

```ts
affects(user);
refresh(user);
```

Store forms:

```ts
affects(store);
affects(store.user, "name");
```

Do not pass wrapper functions or already-read values; invalid targets emit
`INVALID_AFFECTS_TARGET`.

## Actions and optimistic state

Use actions for writes that span async work.

```ts
const save = action(async function* (todo: Todo) {
  setOptimistic(s => {
    s.status = "saving";
  });

  const result = await api.save(todo);

  yield; // transaction-aware resumption before later writes

  setOptimistic(s => {
    s.status = "saved";
  });

  return result;
});
```

Calling an action from an ordinary owned computation/component scope emits
`ACTION_CALLED_IN_OWNED_SCOPE`.

A plain `await` does not by itself re-enter the action transaction for later
writes. Use `yield` as the transaction-safe suspension/resumption point.

Do not call `flush()` inside an action body.

### Optimistic signal

```ts
const [name, setName] = createOptimistic("Alice");
```

### Optimistic store

```ts
const [todos, setTodos] = createOptimisticStore(
  () => api.list(),
  []
);
```

Optimistic state represents the value you can already predict. `affects`
represents a value you know will change but cannot yet provide.

## Stores

### Draft-first setter

```ts
const [store, setStore] = createStore({
  user: { name: "A" },
  list: [] as string[]
});

setStore(s => {
  s.user.name = "B";
  s.list.push("x");
});
```

### Returning a replacement

```ts
setStore(s => ({
  ...s,
  list: []
}));
```

For arrays/objects this performs shallow replacement/diff semantics.

### Compatibility path setter

```ts
setStore(
  storePath("user", "address", "city", "Paris")
);
```

Use this mainly for migration compatibility; draft-first is the default 2.0
shape.

### Derived stores

Readonly:

```ts
const users = createProjection(
  async () => api.listUsers(),
  []
);
```

Writable derived store:

```ts
const [cache, setCache] = createStore(
  draft => {
    draft.value = expensive(selector());
  },
  { value: 0 }
);
```

When a projection derive returns data, the result is reconciled into the
projection. The default identity key is `"id"`.

### `reconcile`

```ts
setStore(s => {
  reconcile(serverTodos, "id")(s.todos);
});
```

- omitted key: default `"id"`;
- `null`: positional matching.

### `shallow: true`

A specialized record-replacement optimization.

```ts
const [rows, setRows] = createStore(initialRows, {
  shallow: true
});
```

Below the shallow boundary, records are plain and should be replaced, not
mutated field-by-field.

This is appropriate when records arrive wholesale and profiling shows deep
proxying is unnecessary overhead.

### `merge`, `omit`, `snapshot`, `deep`

```ts
const props2 = merge(defaults, props, overrides);
const rest = omit(props, "class", "style");

const plain = snapshot(store);
const deeplyTrackedPlain = deep(store);
```

`undefined` is a real override value in `merge`.

## Props

Call accessors at the JSX boundary:

```tsx
<Counter value={count()} />
```

Read properties in the child:

```tsx
function Counter(props: { value: number }) {
  return <span>{props.value}</span>;
}
```

Do not destructure reactive props at component top level:

```tsx
// Bad
function Counter({ value }: { value: number }) {
  return <span>{value}</span>;
}
```

If an API intentionally forwards a getter, make that explicit in its name/type.

## Control flow

### `For`

Default / keyed-by-identity:

```tsx
<For each={items()}>
  {(item, i) => (
    <Row item={item} index={i()} />
  )}
</For>
```

- `item`: plain value;
- `i`: accessor.

Non-keyed:

```tsx
<For each={items()} keyed={false}>
  {(item, i) => (
    <Row item={item()} index={i} />
  )}
</For>
```

- `item`: accessor;
- `i`: plain number.

Custom key:

```tsx
<For each={rows} keyed={row => row.id}>
  {row => <Row row={row()} />}
</For>
```

The custom-key form provides a row accessor so same-key replacement can update
the row without recreating the keyed DOM identity.

### `Repeat`

```tsx
<Repeat count={store.items.length}>
  {i => <Row item={store.items[i]} />}
</Repeat>
```

`i` is a plain number.

### `Show`

Function-child narrowing receives an accessor:

```tsx
<Show when={user()} fallback={<Login />}>
  {u => <Profile user={u()} />}
</Show>
```

### `Errored`

The error argument is an accessor in the current RC control-flow API:

```tsx
<Errored
  fallback={(err, reset) => (
    <button onClick={reset}>
      retry {String(err())}
    </button>
  )}
>
  <Page />
</Errored>
```

Treat the fallback as a render callback, not a component constructor.

### `Reveal`

Coordinates sibling `Loading` boundaries.

```tsx
<Reveal collapsed>
  <Loading fallback={<Skeleton />}><A /></Loading>
  <Loading fallback={<Skeleton />}><B /></Loading>
</Reveal>
```

Orders include `"sequential"`, `"together"`, and `"natural"`.

## Context

Default-less context:

```tsx
const Session = createContext<SessionState>();

<Session value={session}>
  <Page />
</Session>;

const session = useContext(Session);
```

No provider -> `ContextNotFoundError`. `useContext` is typed as the concrete
value, so the common 1.x/React wrapper that checks for `undefined` is not
needed.

Context with a default remains useful for primitive fallback values such as
theme or locale.

Intentional app-global signal/store at module scope is valid. Be careful with
SSR/request isolation and unmanaged resources.

## DOM

```ts
const dispose = render(
  () => <App />,
  document.getElementById("root")!
);

hydrate(
  () => <App />,
  document.getElementById("root")!
);
```

Dynamic factory:

```ts
const Active = dynamic(
  () => editing() ? Editor : Viewer
);
```

Wrapper form:

```tsx
<Dynamic
  component={editing() ? Editor : Viewer}
  value={value()}
/>
```

Class array/object forms:

```tsx
<div
  class={[
    "card",
    props.class,
    {
      active: active(),
      invalid: !valid()
    }
  ]}
/>
```

## SSR/hydration options

Async-aware computes can use `ssrSource` policy.

Conceptually:

- `"server"`: serialized server answer is authoritative for initial hydration;
- `"hybrid"`: seed from server, then compute on client;
- `"client"`: client-owned compute, with structural or declared first paint.

Other advanced options include:

- `deferStream: true` for server stream policy where supported;
- `transparent: true` for hydration integration nodes that must not consume a
  normal hydration id slot.

These are integration-tier features. Verify current source before generating
framework-level hydration code.

## Diagnostic/programmatic dev access

```ts
import { DEV } from "solid-js";

const off = DEV?.diagnostics.subscribe(event => {
  console.log(
    event.code,
    event.severity,
    event.message
  );
});

const capture = DEV?.diagnostics.capture();

// code under test

const events = capture?.stop() ?? [];
off?.();
```

`DEV` is development-only from the public `solid-js` entry.

See `dev-diagnostics.md` for the current code union and behavior.

## Upstream baseline

Primary snapshot sources:

- `packages/solid/CHEATSHEET.md`
- `documentation/solid-2.0/01-reactivity-batching-effects.md`
- `documentation/solid-2.0/03-async.md`
- `documentation/solid-2.0/04-stores.md`
- `packages/solid-signals/src/signals.ts`
- `packages/solid-signals/src/core/*.ts`
