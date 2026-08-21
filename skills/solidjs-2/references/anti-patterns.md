# SolidJS 2.0 RC anti-patterns

Snapshot baseline: **`solid-js@2.0.0-rc.1`**, branch `next`, verified
2026-08-21.

This file intentionally separates **runtime/API-backed mistakes** from
**project conventions**. Do not present an opinionated convention as though
Solid itself rejects other designs.

# Runtime/API-backed mistakes

## 1. Passing a signal accessor where a normal prop value is expected

```tsx
// Bad: child receives a function.
<Counter value={count} />

function Counter(props: { value: () => number }) {
  return <span>{props.value()}</span>;
}
```

Prefer:

```tsx
<Counter value={count()} />

function Counter(props: { value: number }) {
  return <span>{props.value}</span>;
}
```

Props are already reactive at property access. Only forward a getter when the
component API intentionally asks for one.

## 2. Destructuring or capturing reactive props at component top level

```tsx
// Bad: one-time untracked capture.
function Counter(props: { value: number }) {
  const value = props.value;
  return <span>{value}</span>;
}
```

```tsx
// Bad for the same reason.
function Counter({ value }: { value: number }) {
  return <span>{value}</span>;
}
```

Prefer:

```tsx
function Counter(props: { value: number }) {
  return <span>{props.value}</span>;
}
```

A normal top-level reactive capture emits `STRICT_READ_UNTRACKED`. If that
untracked read encounters a still-pending async value, dev throws
`PENDING_ASYNC_UNTRACKED_READ`.

Use `untrack` only when the snapshot is deliberate.

## 3. Treating component ownership as dependency tracking

Bad mental model:

```text
inside component => tracked
```

Better:

```text
component body             => owned
JSX / memo / effect compute => tracked
```

`getOwner()` and `getObserver()` are distinct runtime concepts.

## 4. One-argument `createEffect`

```ts
// Bad 1.x shape.
createEffect(() => {
  console.log(count());
});
```

In current 2.0 dev mode this emits `MISSING_EFFECT_FN`.

Use split effects:

```ts
createEffect(
  () => count(),
  value => {
    console.log(value);
  }
);
```

For derived values, use `createMemo`. For a one-shot imperative call, call the
function directly.

## 5. Putting side effects or writes in effect compute

```ts
// Bad: compute is for dependency collection / derivation.
createEffect(
  () => {
    setLastSeen(count());
    return count();
  },
  value => log(value)
);
```

Writes in ordinary owned computation are guarded by
`REACTIVE_WRITE_IN_OWNED_SCOPE`.

Move writes to the apply phase or another imperative boundary:

```ts
createEffect(
  () => count(),
  value => {
    setLastSeen(value);
    log(value);
  }
);
```

## 6. Reading a store proxy for the first time in effect apply

```ts
// Bad: user.name and user.age are read in untracked apply.
createEffect(
  () => store.user,
  user => send(user.name, user.age)
);
```

Read the fields in compute:

```ts
createEffect(
  () => ({
    name: store.user.name,
    age: store.user.age
  }),
  value => send(value.name, value.age)
);
```

For deep subscription, use `deep(store)` in compute.

## 7. Invoking an action from component/computation setup

```ts
// Bad
const result = saveAction();
```

when executed synchronously inside an ordinary owned component/computation
scope.

Current dev mode emits `ACTION_CALLED_IN_OWNED_SCOPE`.

Call actions from imperative scopes such as event handlers.

## 8. Assuming plain `await` preserves action transaction semantics for later writes

```ts
const save = action(async function* () {
  const result = await api.save();

  // Bad assumption: this write is automatically back inside the transaction.
  setState(s => {
    s.result = result;
  });
});
```

Use the transaction-aware suspension/resumption point:

```ts
const save = action(async function* () {
  const result = await api.save();

  yield;

  setState(s => {
    s.result = result;
  });
});
```

Use `yield` before writes that occur after an internal `await` when you need
those writes to re-enter the action transaction.

## 9. Calling `flush()` inside an action

An action is already managing transactional flush/settle behavior. A manual
flush can expose a partially completed step.

Do not do it.

## 10. Treating `Loading` as "source is fetching"

Bad model:

```text
request in flight => Loading fallback
```

Current model:

```text
branch has no ready answer yet
  => Loading may show fallback

branch has rendered before + normal revalidation
  => keep stale content

Loading on={key} + key changed + pending
  => boundary may re-show fallback
```

Therefore avoid wrapping an entire stable shell unless the whole shell truly
has one readiness boundary.

```tsx
<AppShell>
  <Loading fallback={<ProfileSkeleton />}>
    <Profile />
  </Loading>
</AppShell>
```

## 11. Asking `isPending` about the wrong expression

```ts
const user = createMemo(() => fetchUser(id()));

// Usually not what you mean:
isPending(id);
```

`id` itself is not the lower async computation.

Read the async value:

```ts
isPending(() => user());
```

`isPending` actively performs the expression's read, so place it under the
boundary that owns initial readiness if that read can itself be not ready.

## 12. Expecting a bare `refresh()` to behave like old `resource.loading`

```ts
refresh(user);
```

A same-question refresh is intentionally quiet. Existing data can continue to
answer the question while the fresh result arrives.

If the reload itself should count as pending:

```ts
affects(user);
refresh(user);
```

## 13. Using `loadingValue` as disguised fake data

Bad:

```ts
createMemo(
  () => fetchAccount(),
  {
    loadingValue: {
      name: "Real Customer",
      balance: 999999
    }
  }
);
```

`loadingValue` is a committed first value, not a fallback tree and not an
excuse to impersonate a server answer.

Use a clearly provisional value:

```ts
createMemo(
  () => fetchAccount(),
  {
    loadingValue: {
      provisional: true,
      name: "",
      balance: 0
    }
  }
);
```

or use `<Loading>`.

## 14. Creating nested primitives inside `createTrackedEffect` or owner-backed `onSettled`

```ts
// Bad
onSettled(() => {
  const [value, setValue] = createSignal(0);
});
```

These are restricted leaf scopes. Current dev mode emits
`PRIMITIVE_IN_FORBIDDEN_SCOPE`.

Create the primitive in the parent owner:

```ts
const [value, setValue] = createSignal(0);

onSettled(() => {
  console.log(value());
});
```

## 15. Calling `onCleanup` inside those restricted leaf scopes

```ts
// Bad
onSettled(() => {
  onCleanup(() => teardown());
});
```

Dev emits `CLEANUP_IN_FORBIDDEN_SCOPE`.

Return cleanup directly:

```ts
onSettled(() => {
  const resource = setup();
  return () => resource.dispose();
});
```

## 16. Returning cleanup from an unowned/out-of-band `onSettled`

An `onSettled` scheduled without an owner cannot attach a returned cleanup to a
lifecycle.

Current dev mode emits `SETTLED_CLEANUP_UNOWNED`.

If the work owns a resource, schedule it from an owned component/custom
primitive or explicitly design its disposal.

## 17. Calling `flush()` from effect callbacks as though it always forces another drain

Inside a normal effect apply callback, current dev mode emits
`FLUSH_IN_EFFECT_CALLBACK`; the call is a no-op because the effect is already
running inside a flush.

Inside `createTrackedEffect` / owner-backed `onSettled`, a re-entrant
`flush()` currently throws without a stable diagnostic code.

If work truly must drain later:

```ts
queueMicrotask(() => flush());
```

But needing this routinely is usually a design smell.

## 18. Using `sync: true` and returning async work

```ts
createMemo(
  () => fetchUser(),
  { sync: true }
);
```

`sync: true` asserts that the computation is synchronous. Returning a Promise
or AsyncIterable emits `SYNC_NODE_RECEIVED_ASYNC` in dev.

Remove `sync: true` for normal async-aware behavior.

## 19. Creating effects without a lifecycle owner

```ts
// Suspicious at module scope.
createEffect(
  () => count(),
  value => console.log(value)
);
```

Current dev mode warns with `NO_OWNER_EFFECT`.

This is different from an intentionally global signal/store. The problem is
the effect's disposal lifecycle.

## 20. Treating every module-scope signal/store as invalid

This is the inverse mistake.

```ts
// Can be valid when process-global state is intentional.
export const [theme, setTheme] = createSignal("dark");
```

Solid's current guidance explicitly permits module-scope state as app-global
state.

Ask instead:

- should the state be shared for the process lifetime?
- under SSR, must it be request/user isolated?
- does the module also create listeners/effects/resources that need cleanup?

A request-specific auth/session store at module scope in SSR is dangerous.
A static app-wide theme preference may be intentional.

## 21. Re-implementing a default-less context missing-provider wrapper

```ts
const Session = createContext<SessionState>();

// Redundant 1.x/React-style wrapper.
function useSession() {
  const value = useContext(Session);
  if (!value) throw new Error("missing Session");
  return value;
}
```

In the current 2.0 type/runtime model, default-less `useContext(Session)`
already returns `SessionState` and throws when no provider exists.

Use it directly.

## 22. Getting `<For keyed={false}>` callback arguments wrong

Bad:

```tsx
<For each={items()} keyed={false}>
  {(item, i) => (
    <Row item={item()} index={i()} />
  )}
</For>
```

`i` is not an accessor in non-keyed mode.

Correct:

```tsx
<For each={items()} keyed={false}>
  {(item, i) => (
    <Row item={item()} index={i} />
  )}
</For>
```

Remember:

```text
default/keyed:
  item = plain value
  i    = accessor

keyed={false}:
  item = accessor
  i    = plain number

Repeat:
  i    = plain number
```

## 23. Treating `Errored`'s error argument as a plain error

Bad:

```tsx
<Errored
  fallback={(err, reset) => (
    <button onClick={reset}>
      {String(err)}
    </button>
  )}
>
```

Current RC control flow passes an error accessor.

Correct:

```tsx
<Errored
  fallback={(err, reset) => (
    <button onClick={reset}>
      {String(err())}
    </button>
  )}
>
```

Also treat this `fallback` as a render callback, not a component constructor.

## 24. Passing invalid `affects()` targets

Bad:

```ts
affects(() => user());
affects(user());
```

`affects` expects the original source accessor or a store node.

Use:

```ts
affects(user);
affects(store);
affects(store.user, "name");
```

Invalid shapes emit `INVALID_AFFECTS_TARGET`.

## 25. Inventing dev diagnostic identifiers

The current runtime publishes a typed `DiagnosticCode` union.

Do not convert an uncoded dev throw into a made-up symbolic code.

Examples of uncoded conditions in the current source include:

- invalid cleanup return from an effect/tracked-effect callback;
- re-entrant `flush()` throw inside `createTrackedEffect` / owner-backed
  `onSettled`.

Quote the actual message or describe the condition unless the source exposes a
stable code.

# Project conventions — not Solid rules

The following may be good architecture choices, but do **not** label them as
framework errors.

## Tuple `[state, actions]` return shape

A custom primitive may use `[state, actions]` because it communicates read/write
roles clearly. A flat object is not inherently invalid Solid code.

Use the tuple form as a project convention when it improves consistency.

## One derived pipeline instead of several stores

Combining server data and overlays in a derived/optimistic store can reduce
manual synchronization. Multiple stores are not automatically a bug.

Choose based on whether state has one lifecycle/identity model or genuinely
independent concerns.

## Prop drilling vs Context

There is no official "two hops", "three hops", or other threshold.

Use props when explicit local data flow is clearer. Use Context when the state
is naturally scoped to a subtree with distributed consumers.

## Controlled vs uncontrolled inputs

Both are valid. Avoid creating reactive state only because React muscle memory
tells you every field must be controlled, but use a signal when other reactive
consumers genuinely need the value.

## Class arrays/objects vs manual strings

`class={[...]}` / object forms are excellent 2.0 ergonomics, but a computed
class string is not a runtime violation. Prefer the native structured form
when it is simpler.

## Bulk API calls

Prefer a real backend bulk endpoint when one exists. Do not invent
`api.archiveMany()` merely to satisfy a frontend "anti-pattern" rule.

This is an API architecture concern, not a Solid semantic requirement.

# Review heuristic

When reviewing suspicious code, ask in this order:

1. Where is the reactive read?
2. Is that location tracked?
3. Is a write/action happening inside ordinary owned computation?
4. Can this async read be not ready, and who owns that readiness?
5. Is `Loading` initial branch readiness or intentionally re-armed?
6. Does `isPending` read the async source itself?
7. Who owns cleanup?
8. Is a callback a component, a render callback, or an effect apply phase?
9. What exact dev diagnostic does current source emit?

Prefer evidence from current dev mode over intuition.
