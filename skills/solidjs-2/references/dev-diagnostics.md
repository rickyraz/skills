# SolidJS 2.0 dev-mode diagnostics

Solid 2.0 ships a structured diagnostics system: every dev-mode mistake gets
a stable code, a severity, and an actionable message. Errors throw and halt
execution; warnings log and let execution continue. All of it strips out of
production builds. When something behaves unexpectedly, check the console
for one of these codes before guessing - the message almost always names the
exact problem and the fix.

## Errors (throw in dev)

### `SIGNAL_WRITE_IN_OWNED_SCOPE`

Writing to a signal from inside a reactive scope - a component body, a
memo, an effect's compute function - throws. This is what stops accidental
feedback loops (write triggers a re-read that triggers another write).

```ts
// Throws
createMemo(() => setCount(count() + 1));

// Fix: derive instead of writing back
const doubled = createMemo(() => count() * 2);

// Fix: write from an event handler instead
button.onclick = () => setCount((c) => c + 1);
```

If a signal is genuinely internal to a primitive you're authoring and needs
to write from within its own owned scope, opt in narrowly:
`createSignal(initial, { ownedWrite: true })`. This is an escape hatch for
library authors, not something to reach for to silence the error on app
state.

### `PENDING_ASYNC_UNTRACKED_READ`

Reading an async value that hasn't resolved yet, outside a tracked scope
(component body top level, an effect callback outside its compute phase),
throws - the runtime has nowhere to suspend the read to.

```tsx
// Throws if user() is async and still pending
function Bad() {
  const name = user().name;
  return <div>{name}</div>;
}

// Fix: read inside JSX, where the compiler tracks it
function Good() {
  return <div>{user().name}</div>;
}
```

### `CLEANUP_IN_FORBIDDEN_SCOPE`

`onCleanup` cannot be called from inside `createTrackedEffect` or
`onSettled` - both manage cleanup exclusively through their return value.

```ts
// Throws
onSettled(() => {
  onCleanup(() => teardown());
});

// Fix: return the cleanup function directly
onSettled(() => {
  const id = setInterval(tick, 1000);
  return () => clearInterval(id);
});
```

### Nested primitive creation inside a forbidden scope

`createTrackedEffect` and `onSettled` run as leaf owners - you cannot create
new signals, memos, or effects inside them.

```ts
// Throws
onSettled(() => {
  const [s, setS] = createSignal(0);
});

// Fix: create primitives in the component body, use them inside onSettled
const [s, setS] = createSignal(0);
onSettled(() => console.log(s()));
```

### Invalid cleanup return value

Effect, tracked-effect, reaction, and `onSettled` callbacks must return
either a cleanup function or `undefined` - returning anything else (a
number, an object, a string) throws.

### `flush()` inside a forbidden scope

Calling `flush()` from inside `createTrackedEffect` or `onSettled` would
re-enter the flush cycle while it's already running. Schedule that work
outside those scopes instead.

### Potential infinite loop

Fires when the flush cycle exceeds 100,000 iterations in one tick - almost
always a write that triggers a read that triggers the same write again.
Trace the write chain rather than assuming it's a framework bug.

## Warnings (console.warn in dev, execution continues)

### `STRICT_READ_UNTRACKED`

A signal, signal-backed prop, or store property was read at the top level of
a component body (or inside an effect callback outside its tracked phase).
The value is captured once and will never update from that read.

```tsx
// Warns: captured once, never updates
function Bad(props: { count: number }) {
  const n = props.count;
  return <div>{n}</div>;
}

// Fix: read inside JSX so it stays tracked
function Good(props: { count: number }) {
  return <div>{props.count}</div>;
}

// Fix: if a one-time read is actually intended, say so explicitly
function AlsoGood(props: { count: number }) {
  const n = untrack(() => props.count);
  return <div>{n}</div>;
}
```

### `ASYNC_OUTSIDE_LOADING_BOUNDARY`

An async read happened with no `Loading` ancestor to catch it. This is not
an error: `render()`/`hydrate()` defer attaching the root DOM until the
uncaught async settles, then attach atomically - the mount point just stays
empty (or keeps its existing static content) in the meantime. Treat the
warning as an FYI that you may want an explicit `Loading` boundary for
fallback UI or partial progressive mount, not as something broken. This only
fires during the synchronous body of `render()`/`hydrate()` - later route
transitions run under their own transition and don't trigger it.

### `PENDING_ASYNC_FORBIDDEN_SCOPE`

An async value was read inside `createTrackedEffect` or `onSettled`. These
scopes can't suspend, so if that value is ever actually pending at runtime
this will throw - use a split `createEffect` instead, which is async-aware.

### `NO_OWNER_EFFECT`

An effect was created with no parent owner (usually: at module scope, or
after its owner was already disposed) - it will run forever and never be
cleaned up.

```ts
// Warns: no owner to dispose this
createEffect(() => count(), (v) => console.log(v));

// Fix: create it inside a component or createRoot
createRoot(() => {
  createEffect(() => count(), (v) => console.log(v));
});
```

### `NO_OWNER_CLEANUP`

`onCleanup` was called with no active owner - the cleanup function will
never run.

### `NO_OWNER_BOUNDARY`

A `Loading` or `Errored` boundary was created with no parent owner, so it
will never be disposed.

### `RUN_WITH_DISPOSED_OWNER`

`runWithOwner` was called with an owner that's already disposed - anything
created inside will leak, since nothing will ever dispose it.

## Quick reference table

| Code                             | Severity | Trigger                                                        |
| --------------------------------- | -------- | ----------------------------------------------------------------- |
| `SIGNAL_WRITE_IN_OWNED_SCOPE`     | error    | Signal write inside a component/computation                       |
| `PENDING_ASYNC_UNTRACKED_READ`    | error    | Reading pending async outside a tracking scope                    |
| `CLEANUP_IN_FORBIDDEN_SCOPE`      | error    | `onCleanup` inside `createTrackedEffect`/`onSettled`               |
| `ASYNC_OUTSIDE_LOADING_BOUNDARY`  | warn     | Async read with no `Loading` ancestor (non-halting)                |
| `STRICT_READ_UNTRACKED`           | warn     | Untracked reactive read in a component/effect body                 |
| `PENDING_ASYNC_FORBIDDEN_SCOPE`   | warn     | Pending async read inside `createTrackedEffect`/`onSettled`        |
| `NO_OWNER_EFFECT`                 | warn     | Effect created with no reactive owner                              |
| `NO_OWNER_CLEANUP`                | warn     | `onCleanup` called with no owner                                   |
| `NO_OWNER_BOUNDARY`               | warn     | Boundary (`Loading`/`Errored`) created with no owner                |
| `RUN_WITH_DISPOSED_OWNER`         | warn     | `runWithOwner` called with an already-disposed owner               |

## Programmatic access

For tooling or tests, diagnostics can be observed rather than just logged:

```ts
import { DEV } from "solid-js";

// Live subscription
const unsubscribe = DEV.diagnostics.subscribe((event) => {
  console.log(`[${event.severity}] ${event.code}: ${event.message}`);
});

// Scoped capture, e.g. inside a test
const capture = DEV.diagnostics.capture();
// ...code under test...
const events = capture.stop();
```

Each event carries `code`, `kind` (`"strict-read"` | `"async"` | `"write"` |
`"lifecycle"` | `"owner"`), `severity`, `message`, and, where relevant,
`ownerId`/`ownerName`/`nodeName` for pinpointing which component or signal
triggered it.
