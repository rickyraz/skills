# SolidJS 2.0 dev-mode diagnostics

Solid 2.0 ships structured development diagnostics for mistakes that would
otherwise produce stale reads, feedback loops, leaked owners, invalid async
reads, or broken lifecycle behavior.

Diagnostics have a severity and, where documented here, a stable code.

* **Errors** throw and stop the current execution path.
* **Warnings** use `console.warn` and allow execution to continue.
* Diagnostics are **development-only** and are stripped from production
  builds.

When Solid behaves unexpectedly, **read the diagnostic code before guessing**.
The message usually identifies both the violated runtime rule and the intended
fix.

Do not silence a diagnostic merely to make the console clean. In Solid 2.0,
these guardrails encode important parts of the reactivity model.

---

## Read this first: owner, tracking, and suspension are different concepts

A large percentage of Solid 2.0 diagnostics become obvious once these three
ideas are kept separate.

### Owned

An **owner** controls lifetime and disposal.

Components and most reactive primitives execute with an owner. Primitives
created under that owner can be disposed with it.

Being owned does **not** mean that reactive reads are tracked.

For example, the top level of a component body has an owner, but a signal read
there is not a reactive subscription.

```tsx
function Counter() {
  const [count] = createSignal(0);

  // Owned, but not tracked.
  const captured = count();

  return <div>{captured}</div>;
}
```

`captured` is just the value observed during component creation.

---

### Tracked

A **tracked scope** records reactive dependencies.

Common tracked reads include:

```tsx
<div>{count()}</div>

const doubled = createMemo(() => count() * 2);

createEffect(
  () => count(),
  value => console.log(value),
);
```

In the split `createEffect` form:

```ts
createEffect(compute, apply);
```

`compute` is tracked.

`apply` is not.

That distinction is intentional: dependency discovery and side effects are
separate phases.

---

### Suspendable / async-aware

A scope may also be allowed to participate in Solid's async graph.

A pending async read must occur somewhere the runtime knows how to suspend,
defer, or propagate readiness from.

Some scopes are deliberately **non-suspendable**, even if they participate in
other parts of the reactive system.

In particular, do not assume:

```text
owned === tracked === suspendable
```

They describe different runtime capabilities.

This distinction explains several diagnostics below.

---

## Debugging protocol

When a Solid 2.0 diagnostic appears:

1. **Read the exact code.**
2. Determine whether it is an **error** or **warning**.
3. Identify which rule was violated:

   * write inside owned computation,
   * untracked read,
   * pending async read,
   * forbidden lifecycle scope,
   * missing/disposed owner,
   * invalid cleanup,
   * flush-cycle re-entry.
4. Move the operation to the scope designed for it.
5. Use escape hatches only when authoring a primitive that intentionally needs
   lower-level runtime behavior.

Do **not** start by adding `untrack`, `flush`, another signal, or a wider
`Loading` boundary. Those can hide the actual architectural mistake.

---

# Errors — throw in development

## `SIGNAL_WRITE_IN_OWNED_SCOPE`

A signal was written while executing inside a scope where Solid expects
computation/setup rather than imperative state mutation.

Common sources include:

* component setup,
* `createMemo`,
* the compute half of `createEffect`,
* similar owned reactive computations.

Do not interpret the name as meaning that every owned callback is tracked.
For example, a component body is owned even though top-level reactive reads
are not tracked.

### Wrong

```ts
createMemo(() => {
  setCount(count() + 1);
  return count();
});
```

The computation reads `count`, writes `count`, and can create a feedback cycle:

```text
read count
    ↓
write count
    ↓
invalidate computation
    ↓
read count
    ↓
...
```

### Fix: derive instead of writing back

```ts
const doubled = createMemo(() => count() * 2);
```

### Fix: mutate from an imperative boundary

```tsx
<button onClick={() => setCount(c => c + 1)}>
  Increment
</button>
```

The general rule is:

```text
reactive computation → derive
event/action         → mutate
```

### Escape hatch: `ownedWrite`

A primitive author may occasionally need a signal that can intentionally be
written from its own owned machinery:

```ts
const [value, setValue] = createSignal(initial, {
  ownedWrite: true,
});
```

Treat `ownedWrite: true` as a **library-author escape hatch**, not a normal
application-state option.

If adding it merely makes an error disappear, the state flow is probably
structured incorrectly.

---

## `PENDING_ASYNC_UNTRACKED_READ`

A pending async value was read outside a scope that can track/suspend that
read.

The important condition is **pending**.

Reading an already-settled value may appear to work, which can make this bug
look intermittent.

### Wrong

```tsx
function Profile() {
  // Component setup is not a tracked JSX read.
  const name = user().name;

  return <div>{name}</div>;
}
```

If `user()` is still pending, the runtime has no tracked consumer to attach
that pending state to.

### Fix: perform the read at a tracked consumer

```tsx
function Profile() {
  return <div>{user().name}</div>;
}
```

Or derive it explicitly:

```ts
const name = createMemo(() => user().name);
```

Then consume the derived value reactively:

```tsx
<div>{name()}</div>
```

### Mental model

Passing an async-derived value through the tree and **reading** it are separate
events.

The important question is not:

> Where was the request created?

It is:

> Where was the pending value actually read?

---

## `CLEANUP_IN_FORBIDDEN_SCOPE`

`onCleanup` was called inside a scope that owns cleanup through its **return
value**, specifically `createTrackedEffect` or `onSettled`.

### Wrong

```ts
onSettled(() => {
  const id = setInterval(tick, 1000);

  onCleanup(() => clearInterval(id));
});
```

### Correct

```ts
onSettled(() => {
  const id = setInterval(tick, 1000);

  return () => clearInterval(id);
});
```

The cleanup belongs directly to the lifecycle operation that created the
resource:

```text
setup
  ↓
return cleanup
```

not:

```text
setup
  ↓
register another cleanup mechanism
```

Use `onCleanup` only in the narrower scopes where reactive cleanup registration
is actually supported.

---

## Nested primitive creation inside a forbidden leaf scope

**Throws in development.**

The supplied reference does not include the literal diagnostic code for this
condition. **Do not invent one.**

`createTrackedEffect` and `onSettled` behave as leaf execution scopes. Do not
create signals, memos, effects, or other owned reactive primitives inside
them.

### Wrong

```ts
onSettled(() => {
  const [state, setState] = createSignal(0);
});
```

### Correct

Create owned primitives before entering the leaf scope:

```ts
const [state, setState] = createSignal(0);

onSettled(() => {
  console.log(state());
});
```

Think:

```text
component/custom primitive
│
├─ createSignal
├─ createMemo
├─ createEffect
│
└─ onSettled       ← leaf
```

not:

```text
onSettled
└─ new reactive graph
```

---

## Invalid cleanup return value

**Throws in development.**

The supplied reference does not include the literal diagnostic code for this
condition. **Do not invent one.**

Callbacks whose return contract is:

```ts
void | (() => void)
```

must return either:

* `undefined`, or
* a cleanup function.

Returning a number, object, string, Promise, or another unrelated value is
invalid.

### Wrong

```ts
onSettled(() => {
  startSomething();

  return 123;
});
```

### Correct

```ts
onSettled(() => {
  const resource = startSomething();

  return () => resource.dispose();
});
```

### Important: do not apply this rule to `createEffect`'s compute phase

The compute function exists specifically to return the value passed into the
apply function:

```ts
createEffect(
  () => userId(),
  id => {
    console.log(id);
  },
);
```

Here `userId()` is a normal computation result, **not a cleanup return**.

The cleanup contract belongs to cleanup-bearing lifecycle/effect callbacks,
not to every function passed into a reactive primitive.

---

## `flush()` inside a forbidden scope

**Throws in development.**

The supplied reference does not include the literal diagnostic code for this
condition. **Do not invent one.**

Calling `flush()` from inside `createTrackedEffect` or `onSettled` attempts to
re-enter the scheduler while the current flush is already executing.

Do not do this:

```ts
onSettled(() => {
  setSomething(1);
  flush();
});
```

Move the imperative synchronization point outside the forbidden scope.

`flush()` is primarily for places that genuinely need synchronous observation,
such as:

* tests,
* imperative DOM interop,
* narrow framework/library glue.

It should not become ordinary component-control flow.

---

## Potential infinite loop

**Throws when the scheduler exceeds 100,000 flush iterations in one tick.**

The supplied reference does not include the literal diagnostic code for this
condition. **Do not invent one.**

This almost always means the reactive graph contains a cycle such as:

```text
computation A reads signal X
        ↓
A writes signal Y
        ↓
computation B reads Y
        ↓
B writes X
        ↓
A runs again
```

Do not assume the scheduler itself is broken.

Trace:

1. which write invalidated the computation,
2. which computation re-ran,
3. which value it wrote,
4. whether that write returns to an earlier dependency.

`SIGNAL_WRITE_IN_OWNED_SCOPE` exists partly to prevent the simplest versions of
this pattern before they become scheduler loops.

---

# Warnings — execution continues

## `STRICT_READ_UNTRACKED`

A signal, signal-backed prop, or store property was read outside tracking and
its current value was captured as plain JavaScript data.

This frequently happens at the top level of a component body.

### Wrong

```tsx
function Counter(props: { count: number }) {
  const count = props.count;

  return <div>{count}</div>;
}
```

The read occurred during component setup.

`count` will not become reactive merely because it is later inserted into JSX.

### Correct: defer the read to JSX

```tsx
function Counter(props: { count: number }) {
  return <div>{props.count}</div>;
}
```

### Correct: derive explicitly

```tsx
function Counter(props: { count: number }) {
  const label = createMemo(() => `Count: ${props.count}`);

  return <div>{label()}</div>;
}
```

### Correct: explicitly request a one-time read

If capturing the current value is intentional:

```tsx
function Counter(props: { count: number }) {
  const initialCount = untrack(() => props.count);

  return <div>{initialCount}</div>;
}
```

`untrack` should communicate intent:

> I deliberately want a snapshot.

Do not use it merely to suppress this warning.

---

## `ASYNC_OUTSIDE_LOADING_BOUNDARY`

An async read occurred without a `Loading` ancestor.

This is a **warning, not a correctness failure**.

During the synchronous body of `render()` / `hydrate()`, Solid can defer
attaching the root DOM until the uncaught async value settles, then attach the
result atomically.

Until then, the mount point may:

* remain empty, or
* preserve existing static/hydrated content.

So this:

```text
async read
   ↓
no Loading
   ↓
root waits
   ↓
async settles
   ↓
root attaches
```

is valid behavior.

Use a `Loading` boundary when you want:

* explicit fallback UI,
* progressive mounting,
* only a subsection of the page to wait.

### Example

```tsx
<AppShell>
  <Loading fallback={<ProfileSkeleton />}>
    <Profile user={user()} />
  </Loading>
</AppShell>
```

Prefer narrow boundaries around the UI that actually consumes the async value.

This warning only applies to the synchronous `render()` / `hydrate()` body.
Later route transitions execute under their own transition machinery.

---

## `PENDING_ASYNC_FORBIDDEN_SCOPE`

An async value was read inside `createTrackedEffect` or `onSettled`.

These scopes cannot suspend.

Treat this warning as a **pre-failure signal**: if the value is actually
pending at runtime, the read cannot be completed normally.

### Wrong shape

```ts
onSettled(() => {
  console.log(user());
});
```

if `user()` may still be pending.

### Prefer an async-aware reactive path

Use a split `createEffect` when the side effect depends on reactive async data:

```ts
createEffect(
  () => user(),
  user => {
    console.log(user);
  },
);
```

The compute phase participates in reactive dependency tracking, while the apply
phase receives the resolved/computed value for the side effect.

---

## `NO_OWNER_EFFECT`

An effect was created without a parent owner.

Typical causes:

* module-scope effect creation,
* creating an effect after its previous owner has already been disposed,
* invoking primitive setup from unrelated asynchronous code with no owner.

### Wrong

```ts
createEffect(
  () => count(),
  value => console.log(value),
);
```

at module scope.

Nothing owns the effect, so nothing knows when to dispose it.

### Correct

Create it under a component/custom primitive owner or an explicit root:

```ts
createRoot(() => {
  createEffect(
    () => count(),
    value => console.log(value),
  );
});
```

For application code, component/custom-primitive ownership is usually
preferable to manually creating roots.

### SSR consequence

Module-level reactive state is especially dangerous under SSR because one
process-level instance may accidentally be shared across multiple requests.

---

## `NO_OWNER_CLEANUP`

`onCleanup` was called when no active owner exists.

The cleanup callback therefore has no lifecycle to attach to and will never
run.

### Wrong mental model

```text
onCleanup(fn)
=
global "run this sometime later"
```

### Correct mental model

```text
owner
└─ resource
   └─ cleanup when that owner disposes/re-runs
```

If there is no owner, there is no disposal event to attach the cleanup to.

Move setup and teardown into an owned component/custom primitive, or use an
appropriate explicit root.

---

## `NO_OWNER_BOUNDARY`

A `Loading` or `Errored` boundary was created without a parent owner.

Boundaries participate in the reactive ownership tree and need a lifecycle.

Without an owner they cannot be disposed correctly.

Do not create component/boundary machinery as a process-level singleton.

---

## `RUN_WITH_DISPOSED_OWNER`

`runWithOwner` received an owner that has already been disposed.

Anything created inside that callback can no longer be attached to a live
lifecycle and may leak.

### Common failure shape

```text
capture owner
    ↓
component disposes
    ↓
async callback fires later
    ↓
runWithOwner(oldOwner, ...)
```

Before preserving an owner for asynchronous work, verify that the architecture
actually requires it.

Prefer keeping work inside Solid's normal reactive/transition lifecycle rather
than manually reviving old ownership contexts.

---

# Diagnostic families

When reading a code, first classify the family.

| Family        | What it means                                           |
| ------------- | ------------------------------------------------------- |
| `strict-read` | Reactive data was read without dependency tracking      |
| `async`       | Async readiness was consumed from the wrong place       |
| `write`       | Reactive state was mutated from a forbidden owned scope |
| `lifecycle`   | Cleanup/setup rules were violated                       |
| `owner`       | Something has no live disposal owner                    |

This gives a useful first question:

```text
What kind of runtime capability am I misusing?
```

before asking:

```text
What syntax should I change?
```

---

# Quick reference

| Diagnostic                               | Severity | Meaning                                                         | Default fix                                                        |
| ---------------------------------------- | -------- | --------------------------------------------------------------- | ------------------------------------------------------------------ |
| `SIGNAL_WRITE_IN_OWNED_SCOPE`            | error    | Signal write during forbidden owned execution                   | Derive in computations; mutate from events/actions                 |
| `PENDING_ASYNC_UNTRACKED_READ`           | error    | Pending async value read without a suspendable tracked consumer | Move the read into JSX/memo/async-aware compute                    |
| `CLEANUP_IN_FORBIDDEN_SCOPE`             | error    | `onCleanup` inside `createTrackedEffect` / `onSettled`          | Return cleanup directly                                            |
| Nested primitive in forbidden leaf scope | error    | Primitive created inside `createTrackedEffect` / `onSettled`    | Create it in the parent owned scope                                |
| Invalid cleanup return                   | error    | Cleanup-bearing callback returned a non-function value          | Return `undefined` or cleanup function                             |
| `flush()` in forbidden scope             | error    | Scheduler flush attempted to re-enter itself                    | Move synchronization outside the scope                             |
| Potential infinite loop                  | error    | Flush cycle exceeded 100,000 iterations                         | Trace cyclic write → invalidate → write chain                      |
| `STRICT_READ_UNTRACKED`                  | warn     | Reactive read captured outside tracking                         | Read in JSX/memo/effect compute, or explicit `untrack`             |
| `ASYNC_OUTSIDE_LOADING_BOUNDARY`         | warn     | Initial async read has no `Loading` ancestor                    | Add narrow `Loading` only if fallback/progressive mount is desired |
| `PENDING_ASYNC_FORBIDDEN_SCOPE`          | warn     | Potentially pending read in non-suspendable scope               | Move dependency to async-aware `createEffect` compute              |
| `NO_OWNER_EFFECT`                        | warn     | Effect has no disposal owner                                    | Create under component/custom primitive/`createRoot`               |
| `NO_OWNER_CLEANUP`                       | warn     | Cleanup has no owner                                            | Register cleanup inside owned lifecycle                            |
| `NO_OWNER_BOUNDARY`                      | warn     | Boundary has no owner                                           | Create it under a live render/reactive owner                       |
| `RUN_WITH_DISPOSED_OWNER`                | warn     | Re-entering an owner after disposal                             | Stop retaining/reusing disposed owners                             |

For error conditions whose literal runtime code is not included in this
reference, preserve the behavioral name above and inspect the actual console
message/source before assigning a code.

**Never fabricate a diagnostic identifier.**

---

# Common diagnosis traps

## Trap 1: treating component setup as tracked

Wrong assumption:

```text
inside component
=
reactive
```

Correct:

```text
component setup
=
owned

JSX / memo / effect compute
=
tracked
```

A component executes once. Its JSX expressions establish the fine-grained
reactive reads.

---

## Trap 2: treating every warning as something that must be eliminated

`ASYNC_OUTSIDE_LOADING_BOUNDARY` is intentionally only a warning.

A `Loading` boundary is a UX decision when you want fallback/progressive
mounting, not a universal requirement for correctness.

---

## Trap 3: using `untrack()` as a warning suppressor

This:

```ts
untrack(() => value());
```

means:

> I explicitly want this read not to become a dependency.

It does **not** mean:

> Make Solid stop complaining.

---

## Trap 4: using `flush()` until tests pass

Writes are microtask-batched by default.

If a test needs the committed value synchronously, `flush()` may be valid.

If application logic repeatedly needs `flush()` to work, inspect the state
flow instead.

---

## Trap 5: fixing async diagnostics by widening `Loading`

A pending read is attached to the place that **consumes** the async value.

Prefer:

```tsx
<AppShell>
  <Sidebar />

  <Loading fallback={<ProfileSkeleton />}>
    <Profile user={user()} />
  </Loading>
</AppShell>
```

over:

```tsx
<Loading fallback={<WholeAppSkeleton />}>
  <App />
</Loading>
```

unless the entire application genuinely has one atomic readiness boundary.

---

## Trap 6: confusing the two halves of `createEffect`

Remember:

```ts
createEffect(
  compute, // tracked dependency collection
  apply,   // side effect, receives result
);
```

Do not move side effects into `compute` merely because that makes a dependency
available.

Do not perform pending async accessor reads casually in the untracked apply
phase either; pass the value through the compute result.

---

# Programmatic diagnostics

Diagnostics can be consumed programmatically in development through `DEV`.

```ts
import { DEV } from "solid-js";
```

## Live subscription

```ts
const unsubscribe = DEV.diagnostics.subscribe(event => {
  console.log(
    `[${event.severity}] ${event.code}: ${event.message}`,
  );
});

// later
unsubscribe();
```

Useful for:

* custom devtools,
* debugging overlays,
* logging adapters,
* framework integration tests.

---

## Scoped capture

Tests can capture diagnostics generated by a specific operation:

```ts
const capture = DEV.diagnostics.capture();

// code under test

const events = capture.stop();
```

Then assert on stable diagnostic properties rather than matching arbitrary
console text.

For example:

```ts
const event = events.find(
  event => event.code === "STRICT_READ_UNTRACKED",
);

expect(event?.severity).toBe("warn");
```

---

## Diagnostic event shape

Events expose:

```ts
{
  code,
  kind,
  severity,
  message,
  ownerId?,
  ownerName?,
  nodeName?,
}
```

`kind` is one of:

```ts
"strict-read"
| "async"
| "write"
| "lifecycle"
| "owner"
```

When owner/node metadata is available, prefer it over guessing which component
caused the diagnostic.

A useful debugging formatter is:

```ts
DEV.diagnostics.subscribe(event => {
  const location = [
    event.ownerName && `owner=${event.ownerName}`,
    event.nodeName && `node=${event.nodeName}`,
  ]
    .filter(Boolean)
    .join(" ");

  console.warn(
    `[Solid:${event.code}] ${event.message}` +
      (location ? ` (${location})` : ""),
  );
});
```

---

# Decision tree

When you see a diagnostic:

```text
Is it about a WRITE?
│
├─ yes → Is the write inside component/memo/effect compute?
│        └─ Move mutation to event/action or derive instead.
│
└─ no
   │
   Is it about a READ?
   │
   ├─ STRICT_READ_UNTRACKED
   │   └─ Move read into JSX/memo/effect compute,
   │      or explicit untrack if snapshot is intentional.
   │
   └─ async/pending
       │
       Is the scope allowed to suspend?
       │
       ├─ no → move dependency into async-aware reactive compute.
       │
       └─ yes
           │
           Need fallback/progressive mount?
           ├─ yes → add a narrow Loading boundary.
           └─ no  → uncaught initial root readiness may be acceptable.
```

For lifecycle diagnostics:

```text
Did this code allocate a resource?
│
├─ onSettled / tracked-effect
│   └─ return cleanup directly
│
└─ custom reactive computation
    └─ use the cleanup mechanism supported by that scope
```

For owner diagnostics:

```text
Who disposes this?
│
├─ component/custom primitive/root → good
└─ nobody → ownership bug
```

---

# Rule of thumb

Solid 2.0 diagnostics are usually telling you that an operation happened at
the wrong **phase**, not merely that the syntax is wrong.

Keep these roles separate:

```text
setup owns
tracked computation derives
JSX consumes
Loading expresses initial readiness
effect apply performs side effects
event/action mutates
cleanup disposes
```

When a diagnostic fires, move the operation to the phase that owns that
responsibility instead of disabling the guardrail.
