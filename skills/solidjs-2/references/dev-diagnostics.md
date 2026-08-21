# SolidJS 2.0 RC dev-mode diagnostics

Snapshot baseline: **`solid-js@2.0.0-rc.1`**, `solidjs/solid` branch `next`,
verified 2026-08-21.

The authoritative current diagnostic union lives in:

`packages/solid-signals/src/core/dev.ts`

## Important: severity is not identical to "throws"

The structured event has:

```ts
type DiagnosticSeverity = "warn" | "error";

type DiagnosticKind =
  | "strict-read"
  | "async"
  | "write"
  | "lifecycle"
  | "owner"
  | "error";
```

An event with `severity: "error"` does **not** guarantee the runtime throws at
that line.

Examples:

- many user-authorship violations emit severity `error` and throw;
- `INVARIANT_VIOLATION` logs in normal dev and throws under test mode;
- `REACTIVITY_HALTED` reports a fatal scheduler state after an uncaught error.

Always document the actual behavior of the specific diagnostic.

## Current `DiagnosticCode` union

```ts
type DiagnosticCode =
  | "STRICT_READ_UNTRACKED"
  | "PENDING_ASYNC_UNTRACKED_READ"
  | "PENDING_ASYNC_FORBIDDEN_SCOPE"
  | "REACTIVE_WRITE_IN_OWNED_SCOPE"
  | "ACTION_CALLED_IN_OWNED_SCOPE"
  | "RUN_WITH_DISPOSED_OWNER"
  | "NO_OWNER_CLEANUP"
  | "CLEANUP_IN_FORBIDDEN_SCOPE"
  | "SETTLED_CLEANUP_UNOWNED"
  | "FLUSH_IN_EFFECT_CALLBACK"
  | "PRIMITIVE_IN_FORBIDDEN_SCOPE"
  | "NO_OWNER_EFFECT"
  | "NO_OWNER_BOUNDARY"
  | "ASYNC_OUTSIDE_LOADING_BOUNDARY"
  | "INVALID_REFRESH_TARGET"
  | "INVALID_AFFECTS_TARGET"
  | "MISSING_EFFECT_FN"
  | "SYNC_NODE_RECEIVED_ASYNC"
  | "REACTIVITY_HALTED"
  | "INVARIANT_VIOLATION";
```

If a name is not in this union, do not invent it.

# Mental model behind diagnostics

## Owner

Lifecycle/disposal context.

```ts
getOwner();
```

## Observer / tracking

Dependency-collecting computation.

```ts
getObserver();
```

A component body can have an owner while a top-level read remains untracked.

## Restricted leaf scope

`createTrackedEffect` and owner-backed `onSettled` are intentionally leaf-like.
They cannot host nested reactive primitives or `onCleanup`.

## Async readiness

A pending async value requires a tracked/readiness-aware consumer. A normal
untracked stale read and a pending untracked read are different diagnostics.

# Diagnostic reference

## `STRICT_READ_UNTRACKED`

**Severity:** warn  
**Typical behavior:** console warning; execution continues.

A reactive signal, signal-backed prop, or store property was read directly in
a strict untracked location such as component setup or effect apply.

```tsx
function Bad(props: { count: number }) {
  const n = props.count;
  return <span>{n}</span>;
}
```

Fix by moving the read into JSX, memo, or effect compute.

```tsx
function Good(props: { count: number }) {
  return <span>{props.count}</span>;
}
```

If the one-time snapshot is intentional:

```ts
const n = untrack(() => props.count);
```

## `PENDING_ASYNC_UNTRACKED_READ`

**Severity:** error  
**Behavior:** emits the diagnostic and throws.

A still-pending async value was read in a strict untracked location.

```tsx
function Bad() {
  const name = user().name;
  return <span>{name}</span>;
}
```

If `user()` is pending, the runtime has no tracked consumer to attach retry /
readiness to.

Read it in a tracked consumer:

```tsx
function Good() {
  return <span>{user().name}</span>;
}
```

The runtime message explicitly points to JSX, memo, or effect compute.

## `PENDING_ASYNC_FORBIDDEN_SCOPE`

**Severity:** warn in the current diagnostic design.  
**Meaning:** a pending async read is being attempted from a scope such as
`createTrackedEffect` / `onSettled` that is not meant to suspend normally.

Treat this as a structural warning: move the potentially pending dependency to
an async-aware tracked compute, usually the compute half of `createEffect`.

```ts
createEffect(
  () => user(),
  value => {
    console.log(value);
  }
);
```

## `REACTIVE_WRITE_IN_OWNED_SCOPE`

**Severity:** error  
**Behavior:** emits and throws for guarded writes.

Covers more than signal setters. Current source uses this code for guarded
reactive state writes and guarded `refresh()` invalidation in ordinary owned
component/computation scope.

```ts
createMemo(() => {
  setCount(count() + 1); // error
  return count();
});
```

Derive instead:

```ts
const doubled = createMemo(() => count() * 2);
```

or mutate from an imperative boundary.

`ownedWrite: true` is an advanced opt-in for a primitive whose own internal
write is intentional.

## `ACTION_CALLED_IN_OWNED_SCOPE`

**Severity:** error  
**Behavior:** emits and throws.

Calling an `action` synchronously from an ordinary owned
component/computation scope is invalid.

Call it from an event/imperative scope.

The guard deliberately allows restricted imperative leaf scopes such as
tracked effects/onSettled.

## `RUN_WITH_DISPOSED_OWNER`

**Severity:** warn.  
**Meaning:** `runWithOwner` was given an owner that has already been disposed.

Anything newly attached there has no healthy lifecycle. Capture owners only
when necessary and check `isDisposed(owner)` in late async callbacks.

## `NO_OWNER_CLEANUP`

**Severity:** warn  
**Behavior:** `onCleanup` has no owner to attach to, so it cannot run later.

Move registration under a live owner.

## `CLEANUP_IN_FORBIDDEN_SCOPE`

**Severity:** error  
**Behavior:** emits and throws.

`onCleanup` was called from `createTrackedEffect` / owner-backed `onSettled`.

Return cleanup directly from the callback.

```ts
onSettled(() => {
  const resource = setup();
  return () => resource.dispose();
});
```

## `SETTLED_CLEANUP_UNOWNED`

**Severity:** error  
**Behavior:** emits and throws in dev.

An unowned/out-of-band `onSettled` callback returned a cleanup function, but
there is no owner whose disposal can honor it.

Schedule resource-owning setup from an owned scope or manage its lifecycle
explicitly.

## `FLUSH_IN_EFFECT_CALLBACK`

**Severity:** warn  
**Behavior:** `flush()` is a no-op in a normal effect apply callback because
the flush that runs effects is already active.

Writes from the effect are processed by the running flush continuation.

If an explicit later drain is truly required:

```ts
queueMicrotask(() => flush());
```

Do not confuse this with re-entrant `flush()` from `createTrackedEffect` /
owner-backed `onSettled`: that currently throws a plain Error without this
diagnostic code.

## `PRIMITIVE_IN_FORBIDDEN_SCOPE`

**Severity:** error  
**Behavior:** emits and throws.

A nested reactive owner/primitive was created inside the restricted childless
scope used by `createTrackedEffect` or owner-backed `onSettled`.

Move primitive creation to the parent scope.

## `NO_OWNER_EFFECT`

**Severity:** warn.

An effect has no parent lifecycle owner and therefore will not be disposed
through a normal owner tree.

Module-scope effect creation is a common cause.

This diagnostic does **not** mean that all module-scope signals/stores are
invalid.

## `NO_OWNER_BOUNDARY`

**Severity:** warn.

A `Loading` / `Errored`-style boundary was created outside a live reactive
owner. The boundary has no normal disposal lifecycle.

## `ASYNC_OUTSIDE_LOADING_BOUNDARY`

**Severity:** warn  
**Behavior:** FYI warning during the initial render/hydrate enforcement window.

An uncaught async read outside `Loading` is legal: the root mount can defer
until pending work settles.

Add a `Loading` boundary when you want explicit fallback or progressive
partial mount, not merely to silence this warning.

## `INVALID_REFRESH_TARGET`

**Severity:** error in the public validation path.

`refresh()` was given something that is not a supported refreshable Solid
target.

Pass the original refreshable accessor/store primitive expected by the API,
not an already-read value or arbitrary wrapper.

When debugging exact target rules, read the current `refresh` implementation;
this validation may evolve during RC.

## `INVALID_AFFECTS_TARGET`

**Severity:** error  
**Behavior:** emits and throws for validated bad shapes.

Valid forms include:

```ts
affects(sourceAccessor);
affects(store);
affects(storeRecord, "key");
```

Invalid examples include wrapper functions, already-read values, accessor +
property-key combinations, or treating multiple keys as a path.

## `MISSING_EFFECT_FN`

**Severity:** error  
**Behavior:** emits and throws.

The single-argument Solid 1.x effect shape is no longer supported.

```ts
// Invalid
createEffect(() => count());
```

Use:

```ts
createEffect(
  () => count(),
  value => console.log(value)
);
```

## `SYNC_NODE_RECEIVED_ASYNC`

**Severity:** error  
**Behavior:** emits and throws in dev.

A computation/effect created with `sync: true` returned a Promise or
AsyncIterable.

The option asserts that async handling can be skipped. Remove `sync: true` or
return a synchronous value.

## `REACTIVITY_HALTED`

**Severity:** error event / fatal runtime state.

An uncaught error escaped the available error handling and the scheduler halted
instead of continuing with potentially inconsistent partially-applied state.

After this state, further scheduled updates are ignored and the runtime logs
that reactivity was halted.

Treat it like a crash: fix/contain the original error with the appropriate
error boundary or effect error handling.

## `INVARIANT_VIOLATION`

**Severity:** error event.

This is different from normal user-authorship diagnostics. It represents an
internal runtime consistency assertion.

Current source behavior:

- normal dev: emit diagnostic + `console.error`;
- test mode: also throw so the suite/fuzzer fails hard.

If you see this in ordinary app usage on a current RC with a minimal
reproduction, investigate as a possible framework bug.

# Important uncoded dev errors

Not every dev failure currently has a `DiagnosticCode`.

Do not invent a code for these.

## Invalid effect/tracked-effect cleanup return

Effect apply and tracked-effect callbacks must return either:

```ts
undefined
```

or:

```ts
() => void
```

Returning another value currently throws a plain Error with a message about an
invalid cleanup value.

## Re-entrant `flush()` in `createTrackedEffect` / owner-backed `onSettled`

While the queue is already running, calling `flush()` from these callbacks
throws a plain Error explaining that `flush()` is not re-entrant there.

This is **not** `FLUSH_IN_EFFECT_CALLBACK`; that code belongs to normal effect
apply and is only a warning/no-op.

## Post-`await` first read of unresolved async inside an async computation

Current async runtime has a dev-only authored error when an unresolved async
source is first read only after an `await`, because that read cannot establish
the dependency edge needed to retry when the source settles.

The current thrown message does not use a stable `DiagnosticCode`.

Read async reactive dependencies before the first `await`, or restructure them
as inputs.

# Programmatic diagnostics

The public dev entry exposes `DEV`.

```ts
import { DEV } from "solid-js";
```

`DEV` is undefined/non-meaningful outside the development entry, so code that
may run in production should guard it.

## Subscribe

```ts
const unsubscribe = DEV?.diagnostics.subscribe(event => {
  console.log(
    `[${event.severity}] ${event.code}: ${event.message}`
  );
});
```

## Capture

```ts
const capture = DEV?.diagnostics.capture();

// code under test

const events = capture?.stop() ?? [];
```

A capture also exposes `events` while active and `clear()`.

## Event shape

```ts
interface DiagnosticEvent {
  sequence: number;
  code: DiagnosticCode;
  kind: DiagnosticKind;
  severity: "warn" | "error";
  message: string;
  ownerId?: string;
  ownerName?: string;
  nodeName?: string;
  data?: Record<string, unknown>;
}
```

Use structured `code`, `kind`, and owner/node metadata in tests and tooling.
Do not assert long human message strings unless the wording itself is the
contract you are testing.

# Debugging decision tree

```text
Have an exact diagnostic code?
│
├─ yes
│  ├─ strict-read
│  │  └─ move reactive read into tracked JSX/memo/effect compute,
│  │     or use explicit untrack for an intentional snapshot
│  │
│  ├─ write
│  │  └─ move mutation/action/invalidation to an imperative phase
│  │
│  ├─ async
│  │  └─ identify the actual pending source and the consumer/boundary
│  │
│  ├─ lifecycle / owner
│  │  └─ answer "who owns and disposes this?"
│  │
│  └─ error
│     └─ determine whether this is a fatal halt or internal invariant
│
└─ no
   └─ do not invent a name; use the actual thrown message and inspect source
```

# Source-of-truth rule

When this reference and the current repo disagree, trust
`packages/solid-signals/src/core/dev.ts` and the call site that emits the
diagnostic.

The union is intentionally machine-readable. Re-check it whenever the RC
version changes.
