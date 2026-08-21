---
name: solidjs-2
description: SolidJS 2.0 RC coding guidance grounded in the official `solidjs/solid` `next` branch. Use for writing, reviewing, migrating, or debugging Solid 2.0 code without falling back to React or Solid 1.x assumptions.
---

# SolidJS 2.0

## Baseline and source hierarchy

This skill targets the Solid repository's `next` branch as verified on
**2026-08-21**, where `packages/solid/package.json` declares
**`solid-js@2.0.0-rc.1`**.

Solid 2.0 is still moving. When behavior in this skill conflicts with the
current repository, prefer sources in this order:

1. current implementation in `solidjs/solid` `next`;
2. `packages/solid/CHEATSHEET.md`;
3. `documentation/solid-2.0/*.md`;
4. this skill.

Do not silently combine Solid 1.x docs, an older beta, or React conventions
with this RC snapshot.

## Before writing code: three priors to reject

**React prior:** components do not re-render as the unit of reactivity. Solid
tracks reads and updates the concrete consumers of those reads.

**Solid 1.x prior:** several familiar APIs have changed or disappeared:
`createResource`, `batch`, one-argument `createEffect`, `onMount`,
`createComputed`, `Index`, `Suspense`, `SuspenseList`, `mergeProps`,
`splitProps`, and old store path setters are not the default 2.0 model.

**"Everything inside a component is tracked" prior:** ownership and dependency
tracking are distinct.

## Mental model: owner != observer != async readiness

Keep these concepts separate.

### Owner

An owner is a lifecycle/disposal node. `getOwner()` answers "what owns
primitives and cleanup created here?"

A component body has an owner.

### Observer / tracked computation

A tracked computation records reactive reads. `getObserver()` is the low-level
runtime distinction.

Common tracked locations:

- JSX reactive expressions;
- `createMemo` compute functions;
- the compute half of `createEffect`;
- derived store/projection computes.

The top level of a component body is **owned but not a tracked read site**.

```tsx
function Bad(props: { name: string }) {
  const name = props.name; // captured once; dev warning
  return <h1>{name}</h1>;
}

function Good(props: { name: string }) {
  return <h1>{props.name}</h1>; // JSX read stays reactive
}
```

### Async readiness

A pending async value also needs a consumer through which the runtime can
represent "not ready".

A normal untracked reactive read warns with `STRICT_READ_UNTRACKED`. If the
same kind of read hits a still-pending async value, dev mode throws
`PENDING_ASYNC_UNTRACKED_READ`.

Treat "owned", "tracked", and "async-aware" as separate runtime capabilities,
not interchangeable labels.

## Core rules

### Components run once; reactive reads update consumers

Do not explain Solid as "rerender the component when state changes".
Explain which read is tracked and which DOM/computation depends on it.

### Props are reactive values

Call a signal accessor at the JSX call site and read the prop as a property in
the child.

```tsx
const [count, setCount] = createSignal(0);

<Counter value={count()} />;

function Counter(props: { value: number }) {
  return <span>{props.value}</span>;
}
```

Do not pass `count` unless the API intentionally asks for an accessor.

Do not destructure reactive props at component top level.

### Writes are queued

Setters queue updates. A read immediately after a setter sees the last
committed value until the next microtask flush or explicit `flush()`.

```ts
setCount(1);
count(); // previous committed value
flush();
count(); // 1
```

`batch()` is gone. Use `flush()` only when synchronous observation is actually
required.

### `createEffect` is split

The supported shape separates dependency collection from imperative work.

```ts
createEffect(
  () => ({ id: user.id, enabled: flags.enabled }),
  value => {
    sendAnalytics(value);
    return () => teardown();
  }
);
```

- compute: tracked;
- apply: untracked, imperative, writable;
- apply may return cleanup;
- a one-argument `createEffect(fn)` is invalid and emits
  `MISSING_EFFECT_FN`.

If the effect depends on store properties, read those properties in compute
and pass plain data to apply. Use `deep(store)` when deep observation is truly
needed.

### Do not write or invoke actions from ordinary owned computations

Signal/store writes and `refresh()` inside ordinary owned computation/component
scope are guarded by `REACTIVE_WRITE_IN_OWNED_SCOPE`.

Invoking an `action` there is guarded separately by
`ACTION_CALLED_IN_OWNED_SCOPE`.

Writes/actions belong at imperative boundaries such as event handlers, action
workflows, and effect apply phases. `ownedWrite: true` is a narrow primitive
authoring escape hatch, not an application-state default.

### Async is first-class computation

There is no `createResource` in the 2.0 model.

```ts
const user = createMemo(() => fetchUser(id()));
```

The request can start where it makes architectural sense; the UI readiness
boundary can be lower, near the actual consumer.

```tsx
<AppShell>
  <Loading fallback={<ProfileSkeleton />}>
    <Profile user={user()} />
  </Loading>
</AppShell>
```

`Loading` expresses **branch readiness**, not "show a spinner whenever the
source is fetching".

After content has rendered once, normal revalidation keeps the previous
content visible. `on={key}` can re-arm the boundary so fallback appears again
when that key changes while the branch is pending.

Use `isPending(() => user())` for in-flight change UI. The expression must
reach the async source whose pending state you care about; observing only an
upstream `id()` cannot infer that a lower async consumer is pending.

A bare `refresh(user)` is a quiet re-ask of the same question. Pair
`affects(user)` with `refresh(user)` when the reload itself should read as
pending.

### `loadingValue` is not a `Loading` fallback

`loadingValue` gives a memo a real committed first value. During the first
flight, readers do not suspend and `isPending` stays false.

Use it only when the provisional value is honestly shaped as the value's type
and can be identified as provisional. Otherwise use structural `Loading`.

### Actions are transaction workflows

Use `action` when writes span async work, especially with optimistic state.

```ts
const save = action(async function* (draft: Draft) {
  setOptimistic(s => {
    s.status = "saving";
  });

  const result = await api.save(draft);

  yield; // re-enter the transaction before later writes
  setOptimistic(s => {
    s.status = "saved";
  });

  return result;
});
```

`yield` is the transaction-aware suspension point. A plain `await` inside an
async generator does not itself re-enter the transaction for writes that
follow it; insert a bare `yield` before those writes when needed.

Do not call `flush()` inside action bodies.

### Stores are draft-first

```ts
const [state, setState] = createStore({
  user: { name: "A" },
  todos: [] as Todo[]
});

setState(s => {
  s.user.name = "B";
  s.todos.push(newTodo);
});
```

Returning a value from a setter performs shallow replacement/diff semantics.

Use `createProjection(fn, seed)` for readonly derived stores and
`createStore(fn, seed)` when a derived store also needs imperative writes.

`reconcile(value, key?)` performs identity-preserving reconciliation.
The default key is `"id"`; `null` means positional reconciliation.

`shallow: true` is a specialized record-replacement optimization, not the
default recommendation.

### `For` has two call shapes

Default / keyed mode:

```tsx
<For each={items()}>
  {(item, i) => <Row item={item} index={i()} />}
</For>
```

- `item`: plain value;
- `i`: accessor.

Non-keyed mode:

```tsx
<For each={items()} keyed={false}>
  {(item, i) => <Row item={item()} index={i} />}
</For>
```

- `item`: accessor;
- `i`: plain number.

`Repeat` also gives a plain numeric index.

### Context is for subtree scope

Default-less context already throws if no provider exists, so do not generate
a React-style `useX()` wrapper whose only job is to throw on missing context.

```tsx
const Session = createContext<SessionState>();

<Session value={session}>
  <Page />
</Session>;

const session = useContext(Session);
```

A module-scope signal/store can intentionally be app-global. That is not
automatically an anti-pattern. The real questions are lifetime and isolation:

- is process-global lifetime intentional?
- is this safe under SSR and multiple concurrent requests?
- does anything attached to it require cleanup?

Module-scope effects and unmanaged listeners are much more suspicious because
they have no normal owner/disposal lifecycle.

### Lifecycle

Use `onSettled` for ordinary component-level setup + disposal.

```ts
onSettled(() => {
  const id = setInterval(tick, 1000);
  return () => clearInterval(id);
});
```

`createTrackedEffect` and owner-backed `onSettled` are restricted leaf scopes:

- do not create child reactive primitives inside them;
- do not call `onCleanup` inside them;
- return cleanup directly.

An **unowned** `onSettled`, such as one scheduled from an event/out-of-band
scope, may run but cannot return an owner-bound cleanup. Doing so emits
`SETTLED_CLEANUP_UNOWNED`.

## Diagnostics are part of the model

Before guessing at unexpected behavior, read the dev diagnostic.

Do not invent diagnostic identifiers. The current RC has a typed
`DiagnosticCode` union and programmatic `DEV.diagnostics`.

Important examples:

- `STRICT_READ_UNTRACKED`
- `PENDING_ASYNC_UNTRACKED_READ`
- `REACTIVE_WRITE_IN_OWNED_SCOPE`
- `ACTION_CALLED_IN_OWNED_SCOPE`
- `MISSING_EFFECT_FN`
- `PRIMITIVE_IN_FORBIDDEN_SCOPE`
- `CLEANUP_IN_FORBIDDEN_SCOPE`
- `SETTLED_CLEANUP_UNOWNED`
- `FLUSH_IN_EFFECT_CALLBACK`
- `ASYNC_OUTSIDE_LOADING_BOUNDARY`

Read `references/dev-diagnostics.md` for the complete current union and
behavioral notes.

## Codegen workflow

When asked to write or review Solid 2.0 code:

1. identify every reactive read and where it is tracked;
2. separate pure compute from writes/side effects;
3. identify async sources and where they are consumed;
4. put `Loading` around the branch whose initial readiness should block;
5. use `isPending` only on expressions that reach the pending source;
6. choose store granularity from the data model rather than from React habits;
7. verify `For` callback shape before writing `item()` or `i()`;
8. check lifecycle ownership for effects/listeners/cleanup;
9. run dev mode and use diagnostics as evidence;
10. if an API or behavior is uncertain, verify the current `next` source.

## Migration map

| Solid 1.x instinct | Solid 2.0 RC direction |
| --- | --- |
| `createResource` | async `createMemo` / derived computation + `Loading` |
| `Suspense` | `Loading` |
| `SuspenseList` | `Reveal` |
| `batch` | automatic microtask batching; `flush()` for explicit sync drain |
| `onMount` | `onSettled` |
| one-argument `createEffect` | split `createEffect(compute, apply)` |
| `createComputed` | `createMemo`, split `createEffect`, or function-form signal/store |
| `Index` | `<For keyed={false}>` |
| `mergeProps` | `merge` |
| `splitProps` | `omit` |
| `unwrap` | `snapshot` |
| `createSelector` | `createProjection` |
| old path setter | draft setter; `storePath()` only as compatibility helper |

## Reference files

- `references/api-cheatsheet.md` — concrete API shapes and current RC behavior.
- `references/anti-patterns.md` — runtime-backed mistakes plus clearly marked
  conventions that are not Solid rules.
- `references/dev-diagnostics.md` — current diagnostic union, behavior, and
  debugging workflow.

## Upstream files used for this snapshot

- `packages/solid/package.json`
- `packages/solid/CHEATSHEET.md`
- `documentation/solid-2.0/01-reactivity-batching-effects.md`
- `documentation/solid-2.0/02-signals-derived-ownership.md`
- `documentation/solid-2.0/03-async.md`
- `documentation/solid-2.0/04-stores.md`
- `packages/solid-signals/src/core/dev.ts`
- `packages/solid-signals/src/core/core.ts`
- `packages/solid-signals/src/core/effect.ts`
- `packages/solid-signals/src/core/owner.ts`
- `packages/solid-signals/src/core/scheduler.ts`
- `packages/solid-signals/src/core/action.ts`
- `packages/solid-signals/src/core/async.ts`
- `packages/solid-signals/src/signals.ts`
