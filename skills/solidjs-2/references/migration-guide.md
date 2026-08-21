# Migrating a Solid 1.x project to Solid 2.0 RC

Practical steps for upgrading an existing Solid 1.x app to the current
Solid 2.0 release-candidate APIs.

This guide complements:

- `references/api-cheatsheet.md` for API shapes and runtime semantics;
- `references/anti-patterns.md` for common migration mistakes;
- `references/dev-diagnostics.md` for current dev-mode diagnostics.

Solid 2.0 is still pre-stable. Re-check the upstream `solidjs/solid` `next`
branch when an API or diagnostic disagrees with this guide.

> **Version snapshot (2026-08-21):**
> `packages/solid/package.json` on `solidjs/solid@next` declares
> `solid-js@2.0.0-rc.1` and depends on `@solidjs/signals@^2.0.0-rc.1`.
> Prefer the npm `next` tag when you want the current compatible RC set.
> Pin an exact RC only when reproducibility against a specific snapshot matters.

---

## 1. Install the current RC line

```bash
pnpm add solid-js@next @solidjs/web@next vite-plugin-solid@next
```

For later RC updates:

```bash
pnpm update solid-js @solidjs/web vite-plugin-solid
```

Do not copy old beta-era overrides such as an unrelated
`@solidjs/signals` version unless you have verified that the current package
graph actually requires one.

If you need exact reproducibility, pin matching RC versions together instead
of mixing arbitrary beta/RC packages.

---

## 2. Dedupe Solid across the workspace

This matters especially in monorepos, locally linked packages, patched forks,
or packages developed with `pnpm link`.

Multiple copies of Solid's reactive runtime can produce confusing ownership,
context, hydration, or graph behavior.

For Vite projects:

```ts
// vite.config.ts
import { defineConfig } from "vite";
import solid from "vite-plugin-solid";

export default defineConfig({
  plugins: [solid()],
  resolve: {
    dedupe: [
      "solid-js",
      "@solidjs/web",
      "@solidjs/signals",
      "@solidjs/router",
      "@solidjs/meta"
    ]
  }
});
```

Also inspect:

```bash
pnpm why solid-js
pnpm why @solidjs/signals
pnpm why @solidjs/web
```

The goal is not merely "one entry in package.json"; the goal is one compatible
runtime instance in the final dependency graph.

---

## 3. Update `tsconfig.json`

For web apps:

```json
{
  "compilerOptions": {
    "jsx": "preserve",
    "jsxImportSource": "@solidjs/web"
  }
}
```

Solid 2.0 separates renderer-neutral core types from renderer-owned JSX types.

Replace:

```ts
import type { JSX, ComponentProps } from "solid-js";
```

with:

```ts
import type { JSX, ComponentProps } from "@solidjs/web";
```

For renderer-neutral APIs, prefer `Element` from `solid-js`:

```ts
import type { Component, Element } from "solid-js";

type Wrapper = Component<{
  children?: Element;
}>;
```

`solid-js/jsx-runtime` and `solid-js/jsx-dev-runtime` are no longer the web
JSX runtime type entry points.

---

## 4. Fix package import paths

Search the project for old Solid 1.x subpaths.

| Solid 1.x | Solid 2.0 |
| --- | --- |
| `solid-js/web` | `@solidjs/web` |
| `solid-js/store` | `solid-js` |
| `solid-js/h` | `@solidjs/h` |
| `solid-js/html` | `@solidjs/html` |
| `solid-js/universal` | `@solidjs/universal` |

Example:

```ts
// 1.x
import { render } from "solid-js/web";
import { createStore, reconcile } from "solid-js/store";

// 2.0
import { createStore, reconcile } from "solid-js";
import { render } from "@solidjs/web";
```

Do this import cleanup first, but do not mistake the rest of the migration for
a rename-only exercise.

---

## 5. Fix reactivity semantics before feature-level migration

Several 2.0 changes become easier once the new runtime model is clear.

### Components are owned, but component setup is not a tracked read site

This is a migration trap:

```tsx
// Bad: one-time top-level read.
function Counter(props: { count: number }) {
  const count = props.count;

  return <div>{count}</div>;
}
```

Keep the read in JSX:

```tsx
function Counter(props: { count: number }) {
  return <div>{props.count}</div>;
}
```

Or derive explicitly:

```tsx
function Counter(props: { count: number }) {
  const label = createMemo(() => `Count: ${props.count}`);

  return <div>{label()}</div>;
}
```

If a one-time snapshot is intentional:

```ts
const initialCount = untrack(() => props.count);
```

During migration, `STRICT_READ_UNTRACKED` is useful evidence that a 1.x pattern
captured reactive data too early.

If the same untracked read reaches a still-pending async value, the current
runtime throws `PENDING_ASYNC_UNTRACKED_READ`.

---

## 6. Migrate batching assumptions

Solid 2.0 batches writes by microtask.

A setter does not immediately change what a read returns:

```ts
const [count, setCount] = createSignal(0);

setCount(1);
count(); // still the previous committed value

flush();
count(); // 1
```

The old instinct to use `batch()` should be removed.

Use `flush()` sparingly, mainly for:

- tests;
- narrow imperative DOM integration;
- places that genuinely require a synchronous settled read.

Do not make `flush()` normal application control flow.

Inside a normal effect apply callback, current dev mode warns with
`FLUSH_IN_EFFECT_CALLBACK` and the call is a no-op because the flush is already
running.

---

## 7. Migrate effects first

This is one of the most important breaking changes.

### `createEffect` is now split

Solid 1.x:

```ts
createEffect(() => {
  document.title = name();
});
```

Solid 2.0:

```ts
createEffect(
  () => name(),
  value => {
    document.title = value;
  }
);
```

Think:

```text
compute
  -> tracked
  -> declares dependencies
  -> returns a value

apply
  -> untracked
  -> performs imperative work
  -> may write
  -> may return cleanup
```

A one-argument `createEffect(fn)` is invalid in the current RC and emits
`MISSING_EFFECT_FN`.

### Cleanup moves naturally to the apply phase

Solid 1.x:

```ts
createEffect(() => {
  const id = setInterval(() => {
    console.log(name());
  }, 1000);

  onCleanup(() => clearInterval(id));
});
```

Solid 2.0:

```ts
createEffect(
  () => name(),
  value => {
    const id = setInterval(() => {
      console.log(value);
    }, 1000);

    return () => clearInterval(id);
  }
);
```

### Store reads must happen in compute

Do not pass a store proxy through compute and first read its fields in apply:

```ts
// Bad
createEffect(
  () => store.user,
  user => {
    save(user.name, user.email);
  }
);
```

Instead:

```ts
createEffect(
  () => ({
    name: store.user.name,
    email: store.user.email
  }),
  user => {
    save(user.name, user.email);
  }
);
```

For deep observation:

```ts
createEffect(
  () => deep(store),
  value => persist(value)
);
```

### Do not write during compute

```ts
// Bad
createMemo(() => {
  setTotal(price() * quantity());
  return total();
});
```

Use derivation:

```ts
const total = createMemo(
  () => price() * quantity()
);
```

Current dev mode guards ordinary owned writes with
`REACTIVE_WRITE_IN_OWNED_SCOPE`.

`ownedWrite: true` is an advanced primitive-authoring escape hatch, not a
migration shortcut.

---

## 8. Replace `onMount` with `onSettled`

Solid 1.x:

```ts
onMount(() => {
  measureLayout();
});
```

Solid 2.0:

```ts
onSettled(() => {
  measureLayout();

  const onResize = () => measureLayout();
  window.addEventListener("resize", onResize);

  return () => {
    window.removeEventListener("resize", onResize);
  };
});
```

For normal component lifecycle, prefer `onSettled` with returned cleanup.

`onCleanup` is now a narrower low-level primitive, mainly useful in
custom-primitive/library ownership code.

### Restricted leaf-scope rule

Owner-backed `onSettled` and `createTrackedEffect` are restricted leaf scopes.

Do not create nested signals/memos/effects inside them:

```ts
// Bad
onSettled(() => {
  const [state, setState] = createSignal(0);
});
```

Do not call `onCleanup` inside them:

```ts
// Bad
onSettled(() => {
  onCleanup(teardown);
});
```

Return cleanup directly.

### Unowned `onSettled`

An out-of-band/unowned `onSettled` may schedule work, but if it returns cleanup
there is no owner to attach that cleanup to.

Current dev mode reports:

```text
SETTLED_CLEANUP_UNOWNED
```

---

## 9. Migrate async data

### `createResource` -> async computations

Solid 1.x:

```ts
const [user] = createResource(id, fetchUser);
```

Solid 2.0:

```ts
const user = createMemo(
  () => fetchUser(id())
);
```

Then put UI readiness around the consumer:

```tsx
<Loading fallback={<ProfileSkeleton />}>
  <Profile user={user()} />
</Loading>
```

This is an important architectural change:

```text
where async work starts
!=
where UI readiness blocks
```

The request can start high in the tree; the `Loading` boundary can stay close
to the consumer.

---

## 10. Understand `Loading` before replacing `Suspense`

`Loading` is not simply "`resource.loading` but renamed".

### Initial readiness

```tsx
<Loading fallback={<Spinner />}>
  <Profile />
</Loading>
```

If the branch cannot produce its initial UI because an async value is not
ready, fallback can render.

### Normal revalidation

Once the branch has rendered successfully, normal revalidation keeps the
existing content visible while the new value is being prepared.

Do not migrate with this false mental model:

```text
every refetch
-> tear down content
-> show Loading fallback
```

That is not the normal 2.0 behavior.

### Explicitly re-arm readiness with `on`

If a key change should make the boundary show fallback again:

```tsx
<Loading
  on={userId()}
  fallback={<ProfileSkeleton />}
>
  <Profile />
</Loading>
```

Use a wide boundary only if the whole subtree genuinely shares one readiness
decision.

---

## 11. Replace `resource.loading` with the correct concept

There is no single universal replacement.

### Initial not-ready UI

Use:

```tsx
<Loading fallback={<Spinner />}>
  ...
</Loading>
```

### In-flight value change

Use:

```ts
isPending(() => user())
```

The expression must actually reach the async source whose pending state matters.

This:

```ts
isPending(userId)
```

does not automatically know that a lower `user()` computation triggered by
`userId()` is pending.

### Quiet revalidation

A bare:

```ts
refresh(user);
```

re-asks the same question and may keep the existing answer visible without
making `isPending(user)` true.

If the reload itself should be declared pending:

```ts
affects(user);
refresh(user);
```

---

## 12. Migrate resource mutation patterns to `action`

A common 1.x mutation flow was:

```text
set loading flag
-> await mutation
-> refetch
-> clear loading flag
```

Solid 2.0 provides transactional actions and optimistic helpers.

```ts
const saveTodo = action(async function* (todo: Todo) {
  setOptimisticTodos(state => {
    state.status = "saving";
  });

  const saved = await api.saveTodo(todo);

  yield;

  setOptimisticTodos(state => {
    state.status = "saved";
  });

  refresh(todos);

  return saved;
});
```

### Important: `await` and `yield` are not interchangeable

Inside an async generator action:

```ts
const saved = await api.saveTodo(todo);
```

waits for the Promise, but does not by itself guarantee that later writes
have re-entered the transaction.

Use:

```ts
yield;
```

before writes that must resume transaction semantics after an internal
`await`.

For a directly yielded promise:

```ts
yield api.saveTodo(todo);
```

the yielded operation itself is the transaction-aware suspension point.

Do not call `flush()` inside an action body.

Current dev mode also rejects invoking an action synchronously from an
ordinary owned component/computation scope with:

```text
ACTION_CALLED_IN_OWNED_SCOPE
```

Invoke actions from imperative boundaries such as event handlers.

---

## 13. Migrate stores

### Draft-first setters

Solid 2.0 preferred:

```ts
setStore(state => {
  state.user.address.city = "Paris";
});
```

Old path-style ergonomics are available only as an explicit compatibility
helper:

```ts
setStore(
  storePath(
    "user",
    "address",
    "city",
    "Paris"
  )
);
```

Prefer draft-first updates for new code.

### `produce` is no longer the normal wrapper

Draft mutation is already the default setter shape.

### `unwrap` -> `snapshot`

```ts
const plain = snapshot(store);

JSON.stringify(plain);
```

`snapshot` returns plain non-reactive data for serialization/interop.

### `mergeProps` -> `merge`

```ts
const merged = merge(
  defaults,
  props,
  overrides
);
```

Important semantic change:

```ts
merge(
  { a: 1, b: 2 },
  { b: undefined }
).b;
// undefined
```

`undefined` is an explicit value, not "skip this property".

### `splitProps` -> `omit`

```ts
const rest = omit(
  props,
  "class",
  "style"
);
```

### Derived stores

Readonly derived collection:

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

### `reconcile`

```ts
setStore(state => {
  reconcile(serverTodos, "id")(
    state.todos
  );
});
```

The default identity key is `"id"`.

Use positional reconciliation when appropriate:

```ts
reconcile(serverRows, null)
```

### `shallow: true`

This is a specialized optimization for record-replacement workloads, not a
default migration target.

Use it when records are replaced wholesale and profiling shows deep store
tracking is unnecessary.

---

## 14. Migrate list control flow carefully

### `Index` -> `For keyed={false}`

Do not assume the callback arguments have the same accessor shape as normal
`For`.

Default / keyed `For`:

```tsx
<For each={items()}>
  {(item, i) => (
    <Row
      item={item}
      index={i()}
    />
  )}
</For>
```

```text
item = plain value
i    = accessor
```

Non-keyed:

```tsx
<For
  each={items()}
  keyed={false}
>
  {(item, i) => (
    <Row
      item={item()}
      index={i}
    />
  )}
</For>
```

```text
item = accessor
i    = plain number
```

Custom identity key:

```tsx
<For
  each={rows}
  keyed={row => row.id}
>
  {row => <Row row={row()} />}
</For>
```

`Repeat` also receives a plain numeric index.

---

## 15. Migrate error boundaries

Solid 1.x:

```tsx
<ErrorBoundary fallback={...}>
  <Page />
</ErrorBoundary>
```

Solid 2.0:

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

The current RC error value in this fallback API is an accessor: use `err()`.

Boundaries heal when their failing branch recovers; do not mechanically port
old `resetErrorBoundaries` machinery.

---

## 16. Replace `SuspenseList` with `Reveal`

```tsx
<Reveal collapsed>
  <Loading fallback={<Skeleton />}>
    <A />
  </Loading>

  <Loading fallback={<Skeleton />}>
    <B />
  </Loading>
</Reveal>
```

Current reveal ordering supports concepts such as:

```text
sequential
together
natural
```

Verify the current control-flow API before migrating complex SuspenseList
coordination logic one-for-one.

---

## 17. Migrate Context

### `Context.Provider` -> context-as-provider

Solid 1.x:

```tsx
<SessionContext.Provider value={session}>
  <Page />
</SessionContext.Provider>
```

Solid 2.0:

```tsx
<SessionContext value={session}>
  <Page />
</SessionContext>
```

### Default-less context already handles missing providers

```ts
const SessionContext =
  createContext<SessionState>();

const session =
  useContext(SessionContext);
```

Do not mechanically keep a React-style wrapper whose only job is:

```ts
if (!context)
  throw new Error("missing provider");
```

for a default-less context.

### Module-scope state is not automatically invalid

An intentional application-global signal/store is valid:

```ts
export const [theme, setTheme] =
  createSignal("dark");
```

The migration question is lifetime and isolation:

- should this state be global for the whole process?
- under SSR, should it be per request/user instead?
- are there effects/listeners/resources attached that need disposal?

Module-scope effects are much more suspicious than module-scope state because
they have no normal parent disposal owner.

---

## 18. Migrate DOM directives and refs

### `use:` directives -> ref directive factories

Old:

```tsx
<input use:autofocus />
```

Current direction:

```tsx
<input ref={autofocus} />
```

Or composed refs:

```tsx
<button
  ref={[
    autofocus,
    tooltip({ content: "Save" })
  ]}
/>
```

### Ref callbacks are unowned

Do not port code like this:

```tsx
// Bad migration shape
<div
  ref={el => {
    setup(el);
    onCleanup(() => teardown(el));
  }}
/>
```

The ref callback itself is not where component lifecycle ownership should be
registered.

Keep setup/cleanup in an owned component scope via `onSettled`, or use the
owned setup phase of a directive factory.

### `classList` -> structured `class`

```tsx
<div
  class={{
    active: active(),
    disabled: disabled()
  }}
/>
```

or:

```tsx
<div
  class={[
    "card",
    props.class,
    {
      active: active()
    }
  ]}
/>
```

Previously tolerated `class:` / `style:` namespaces are not the 2.0 structured
binding model.

### Removed namespaces

Do not mechanically preserve:

```text
attr:
bool:
on:
oncapture:
```

Use normal attributes, camelCase event handlers, and native listener/ref
patterns where options are needed.

---

## 19. Remove `/*@once*/` usage intentionally

Do not replace `/*@once*/` with another JSX marker.

Most cases should simply stay reactive:

```tsx
<Component value={props.value} />
```

For DOM initial/default state:

```tsx
<input
  defaultValue={props.initialValue}
/>
```

If a one-time JavaScript read is genuinely required:

```ts
const initial =
  untrack(() => props.value);
```

Use this narrowly.

---

## 20. Full migration map

### Renames / direct conceptual replacements

| Solid 1.x | Solid 2.0 |
| --- | --- |
| `Suspense` | `Loading` |
| `SuspenseList` | `Reveal` |
| `ErrorBoundary` | `Errored` |
| `mergeProps` | `merge` |
| `splitProps` | `omit` |
| `unwrap` | `snapshot` |
| `onMount` | `onSettled` |
| `equalFn` | `isEqual` |
| `getListener` | `getObserver` |
| `Index` | `For keyed={false}` |
| `createSelector` | `createProjection` / function-form `createStore` |
| `createDynamic(source, props)` | `dynamic(source)` factory / `Dynamic` wrapper |
| `classList` | structured `class` object/array forms |

### Removed APIs that require a pattern decision

Do not treat these as pure search-and-replace:

```text
createResource
startTransition
useTransition
batch
createComputed
on
onError
catchError
produce
createMutable
modifyMutable
from
observable
createDeferred
indexArray
resetErrorBoundaries
enableScheduling
writeSignal
```

Typical 2.0 directions:

```text
createResource
-> async createMemo / createProjection / function-form store
-> Loading / isPending / refresh / affects

createComputed
-> createMemo
-> split createEffect
-> function-form createSignal/createStore
depending on intent

batch
-> automatic microtask batching
-> flush only for explicit synchronous settle

Index
-> For keyed={false}

createSelector
-> createProjection

onMount
-> onSettled
```

---

## 21. Dev diagnostics are part of migration

Do not stop at TypeScript errors.

Current RC dev diagnostics catch migration bugs that still compile.

Common ones:

```text
STRICT_READ_UNTRACKED
PENDING_ASYNC_UNTRACKED_READ
REACTIVE_WRITE_IN_OWNED_SCOPE
ACTION_CALLED_IN_OWNED_SCOPE
MISSING_EFFECT_FN
PRIMITIVE_IN_FORBIDDEN_SCOPE
CLEANUP_IN_FORBIDDEN_SCOPE
SETTLED_CLEANUP_UNOWNED
NO_OWNER_EFFECT
ASYNC_OUTSIDE_LOADING_BOUNDARY
INVALID_REFRESH_TARGET
INVALID_AFFECTS_TARGET
SYNC_NODE_RECEIVED_ASYNC
```

Important correction from older beta-era notes:

```text
SIGNAL_WRITE_IN_OWNED_SCOPE
```

is not the current diagnostic name.

Use:

```text
REACTIVE_WRITE_IN_OWNED_SCOPE
```

See `references/dev-diagnostics.md` for the current union and actual behavior.

Do not invent a diagnostic code for a plain dev throw that does not expose one.

---

## 22. Track reactive inputs before `await`

Async computations only establish dependencies from tracked reads that happen
before the asynchronous gap.

Bad:

```ts
const user = createMemo(async () => {
  await delay(200);

  return fetchUser(id());
});
```

Better:

```ts
const user = createMemo(async () => {
  const currentId = id();

  await delay(200);

  return fetchUser(currentId);
});
```

The current runtime has additional dev protection for unresolved async sources
first read only after an `await`, because that shape may not have the dependency
edge required for retry when the source settles.

The migration rule is simple:

```text
read reactive inputs first
-> await
-> use captured inputs
```

---

## 23. Testing migration

Tests that assumed immediate setter visibility may fail:

```ts
setCount(1);

expect(count()).toBe(1); // wrong assumption in 2.0
```

Use:

```ts
setCount(1);
flush();

expect(count()).toBe(1);
```

or await the natural microtask / rendered result.

End-to-end tests often need fewer explicit `flush()` calls because they already
wait for observable UI.

Prefer assertions against public behavior rather than sprinkling `flush()`
everywhere until a test passes.

---

## 24. Ecosystem compatibility

Do not cargo-cult old beta Vite aliases or `noExternal` workarounds.

Before adding compatibility config:

1. check whether the dependency already publishes Solid 2-compatible builds;
2. inspect `pnpm why` for duplicate Solid packages;
3. verify the dependency's current peer dependency range;
4. reproduce the problem with the smallest config;
5. add the narrowest workaround only if still necessary.

An old beta testing-library recipe is not a permanent Solid 2 migration rule.

Also distinguish **Solid 2 core** from **SolidStart** versioning. SolidStart v2
was released on the Solid 1 line and does not imply that an existing SolidStart
application is automatically ready for Solid core 2.0.

---

## 25. Recommended migration order

A practical order that reduces cascading mistakes:

1. **package/import/TS config**
   - renderer packages;
   - JSX ownership;
   - dependency dedupe.

2. **reactivity assumptions**
   - top-level reads;
   - batching;
   - props access.

3. **effects + lifecycle**
   - split `createEffect`;
   - cleanup returns;
   - `onSettled`.

4. **async**
   - `createResource`;
   - `Loading`;
   - `isPending`;
   - `refresh` / `affects`.

5. **actions + optimistic state**
   - mutation workflows;
   - `yield` semantics.

6. **stores**
   - draft-first setters;
   - projections;
   - reconciliation;
   - snapshots.

7. **control flow**
   - `For`;
   - `Index`;
   - `Errored`;
   - `Reveal`.

8. **DOM / directives / refs**

9. **Context and SSR isolation**

10. **dev diagnostic cleanup**

11. **tests and performance verification**

This order matters because later migrations are easier when ownership,
tracking, batching, and async readiness are already understood.

---

## Migration review checklist

Before considering a migrated file complete, verify:

- [ ] imports use the 2.0 package layout;
- [ ] JSX types come from the renderer package;
- [ ] reactive props are not destructured/captured at component setup;
- [ ] no one-argument `createEffect` remains;
- [ ] effect compute contains dependency reads, not imperative writes;
- [ ] effect apply receives the data it needs;
- [ ] lifecycle cleanup is owned and returned from the right callback;
- [ ] async inputs are read before `await`;
- [ ] `Loading` is scoped to actual branch readiness;
- [ ] normal revalidation is not assumed to re-show fallback;
- [ ] `isPending` reads the actual pending source;
- [ ] mutations use an appropriate imperative/action boundary;
- [ ] writes after internal action `await` use transaction-aware `yield` where required;
- [ ] store setters use draft-first semantics;
- [ ] `For` callback accessor/plain-value shapes are correct;
- [ ] `Errored` uses the current error accessor shape;
- [ ] module-scope state is intentionally global and SSR-safe;
- [ ] ref callbacks do not own component cleanup;
- [ ] current dev diagnostics are clean or intentionally understood;
- [ ] old beta workarounds were not preserved without evidence.

---

## Source hierarchy

When this guide disagrees with current Solid code, use this order:

1. current `solidjs/solid` `next` implementation;
2. `packages/solid/CHEATSHEET.md`;
3. `documentation/solid-2.0/MIGRATION.md`;
4. topic-specific `documentation/solid-2.0/*.md`;
5. this local migration guide.

Solid 2.0 is still pre-stable, so migration documentation should be treated as
a versioned snapshot rather than timeless framework behavior.
