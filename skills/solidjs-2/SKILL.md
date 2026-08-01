---
name: solidjs-2
description: Write, review, debug, or migrate SolidJS 2.0 (beta) code - components, signals, createEffect, stores, async data, actions, JSX, and TypeScript config. Consult this BEFORE writing or editing any file that imports from "solid-js" or "@solidjs/*", or whenever the user mentions SolidJS 2.0, "Solid 2.0", "solid-js@next", migrating a Solid 1.x or SolidStart app, createOptimisticStore, the Loading or Errored boundaries, the split-phase createEffect, or any solid-js primitive that changed shape in 2.0. Solid 2.0 is a major breaking rewrite (async-first reactivity, split effects, draft-first stores, removed createResource/Suspense/Index/use:) - code written from general JavaScript, React, or Solid 1.x training data will compile but be subtly wrong. Use this skill even for small snippets; do not rely on prior knowledge of solid-js.
---

# SolidJS 2.0

## Before you write anything: two priors are fighting you

Solid 2.0 departs from **both** React and Solid 1.x. If you've seen more React
than Solid, your first instinct on hooks, effects, props, and mutations will
be React-shaped. If you know Solid 1.x, your instinct on `createEffect`,
`Suspense`, `Index`, and stores will be **also wrong** - 2.0 renamed or
restructured most of it. Treat both as unreliable priors, not sources of
truth. The single most common failure mode isn't missing knowledge, it's
reflexively writing the React- or 1.x-shaped version of something and moving
on. When a pattern feels familiar, that's the moment to check this skill
instead of trusting the reflex.

## Mental model (read first)

A Solid component function runs **once**. JSX reads of signals/props/store
properties compile into fine-grained subscriptions wired directly to DOM
nodes - there is no re-render and no virtual DOM diff. A signal is a pair of
functions `[read, write]`; `read` is a function you call (`count()`), not a
value you hold. Reactivity only happens where a read is *tracked*: inside
JSX, inside `createMemo`, or inside the compute half of `createEffect`. A
read at the top level of a component body, or after an `await`, is untracked
and silently stale - dev mode warns about this, but only warns.

## The big ideas in 2.0

- **Async is first-class.** Any computation (`createMemo`, derived
  `createStore`/`createOptimisticStore`) can return a Promise or async
  iterable. There is no separate `createResource` primitive anymore.
- **Async capability is part of the baseline runtime, not an optional feature.**
  Because promises or async iterators can enter the reactive graph through many
  APIs, Solid 2.0 cannot tree-shake the async machinery the way Solid 1.x could
  gate features behind specific imports. This intentionally raises the minimum
  toy-app runtime size; see `references/runtime-size-and-async-tradeoff.md`.
- **`Loading` handles initial readiness only.** It shows a fallback the
  first time a subtree has nothing to render, then gets out of the way - it
  does not re-trigger on every revalidation the way `Suspense` did.
- **`isPending(() => expr)`** is a read-level "is this specific expression's
  async work still in flight" check, for "refreshing…" UI that doesn't tear
  the screen down.
- **Mutations go through `action()`.** Generator-based: optimistic write,
  `yield` the network call, `refresh()` the derived read.
- **Derived-but-writable state is a primitive**: `createSignal(fn)` and
  `createStore(fn)` make "writable memo" patterns explicit.
- **Batching is automatic and deterministic.** Every write is
  microtask-batched; a read right after a write still returns the old value
  until the microtask flushes (or you call `flush()`).
- **Dev-mode guardrails are load-bearing.** Solid 2.0 throws or warns on a
  fixed set of mistakes (writing inside a reactive scope, untracked reads,
  reading pending async outside a tracking scope). Don't silence these or
  route around them - see `references/dev-diagnostics.md`.

## Quick API map - what replaced what

| Old instinct (React or Solid 1.x)  | Solid 2.0                                                      |
| ----------------------------------- | ---------------------------------------------------------------- |
| `solid-js/web` imports              | `@solidjs/web`                                                  |
| `solid-js/store` imports            | `solid-js` (stores are exported from core now)                  |
| `Suspense`                          | `Loading`                                                       |
| `SuspenseList`                      | `Reveal`                                                        |
| `ErrorBoundary`                     | `Errored`                                                       |
| `createResource`                    | async `createMemo` / derived `createStore` + `Loading`          |
| `resource.loading`                  | `Loading` (initial) / `isPending(() => x())` (revalidation)     |
| `resource.refetch()` / `.mutate()`  | `refresh(x)` / `createOptimisticStore` + `action()`              |
| `startTransition` / `useTransition` | built-in transitions + `isPending` / `Loading` / optimistic APIs |
| `createEffect(fn)` single function  | `createEffect(computeFn, applyFn)` - split, see below            |
| `onMount`                           | `onSettled` (can return a cleanup function)                      |
| `batch(() => { ... })`              | nothing to call - every write batches by default; `flush()` forces sync |
| `Index`                             | `For keyed={false}`                                              |
| `use:directive`                     | `ref={directiveFactory(...)}` (composable via array)             |
| `classList={{ ... }}`               | folded into `class={["base", { toggle: cond() }]}`               |
| `Context.Provider value={v}`        | `Context value={v}` - the context itself is the provider         |
| `mergeProps` / `splitProps`         | `merge` / `omit`                                                 |
| `unwrap(store)`                     | `snapshot(store)`                                                |
| `produce(fn)` wrapper               | not needed - draft-first mutation is the default setter behavior |
| `createMutable`                     | `createStore` with draft setters                                  |
| `createComputed`                    | `createMemo` (pure derive) / split `createEffect` (side effect) / `createSignal(fn)` (writeback) - pick based on which one it was doing |

Full behavior details, before/after code, and the dev-diagnostic table live
in `references/api-cheatsheet.md` - read it before doing anything beyond a
trivial one-line edit.

## The two changes that break the most code

**1. Split `createEffect`.** It's now `createEffect(compute, apply)`:
`compute` runs tracked and returns a value; `apply` receives `(value, prev)`
and may return a cleanup function. Don't perform side effects inside the
tracked half, and don't leave `apply` empty just to keep one function.

```tsx
// Track userId, then separately perform the side effect with its value.
createEffect(
  () => userId(),
  (id) => {
    const controller = new AbortController();
    logVisit(id, controller.signal);
    return () => controller.abort();
  }
);
```

**2. Writes don't update reads until the batch flushes.** `setCount(1);
count()` still returns the old value in that same synchronous block - the
new value appears on the next microtask, or immediately after `flush()`.
This is the most common source of "why didn't my test/log see the update"
confusion. Reach for `flush()` in tests or imperative glue code, never
inside a component just to "make sure" a read is current.

## Common AI mistakes to actively avoid

These are observed, recurring failure patterns, not hypotheticals. Skim
`references/anti-patterns.md` for the full bug-to-fix pairs; the two that
cause the most damage on their own:

- **Passing an accessor as a prop instead of calling it at the JSX call
  site.** `<Row data={data} />` (passing the function itself) is wrong;
  `<Row data={data()} />` is right. Props are reactive *values* read through
  a proxy - a child component should never call `props.x` as a function.
- **Modeling one piece of UI state as several separate stores/signals**
  (server data + error map + saving flags) instead of composing them inside
  a single derived store's projection function, with side channels as plain
  non-reactive objects read inside that projection and invalidated via
  `refresh()`.

Also, by default: don't destructure props, don't reach for Context before
prop-drilling actually gets awkward (roughly 3+ hops or real fan-out), don't
build `class` strings by hand, don't use `use:` directives, don't restate
options that already match their default (e.g. passing `{ key: "id" }` when
`"id"` is the default), and don't create signals or effects at module scope
- there's no owner to dispose them, and on a server they leak across
requests.

## Workflow

**Writing new Solid 2.0 code:** default to an async `createMemo` or derived
store over hand-rolling loading-flag state; default to `Loading` scoped as
narrowly as possible around just the suspending read, not a whole page
shell (it mounts real DOM, not a virtual diff - anything outside the
boundary persists across the transition, anything inside gets rebuilt);
default to draft-mutation store setters; default to props over Context.

**Migrating a Solid 1.x app:** read `references/migration-guide.md` first -
it covers `package.json` / `tsconfig.json` / Vite changes, the `next` npm
tag, and the mechanical rename map. Then work file by file using
`references/api-cheatsheet.md` for the *behavioral* changes in each file,
not just the renames. Don't do a mechanical find-and-replace of old API
names - several aren't 1:1 renames (`createComputed` alone splits three
different ways depending on what it was actually doing).

**Debugging unexpected behavior:** check `references/dev-diagnostics.md` for
the exact error or warning code first - Solid 2.0's dev diagnostics are
specific and name the actual problem, so don't guess before reading it.

## Reference files

- `references/api-cheatsheet.md` - full API surface: imports and tsconfig,
  effects/lifecycle, async (`Loading`, `isPending`, `latest`, `resolve`,
  `refresh`, `affects`), actions/optimistic, stores, control flow (`For`,
  `Show`, `Switch`, `Reveal`, `dynamic`), DOM (attributes/events/`ref`
  directives), and context.
- `references/anti-patterns.md` - concrete bug-to-fix pairs for mistakes
  models commonly make when writing Solid 2.0, with the reasoning behind
  each fix so the correction generalizes past the specific example.
- `references/migration-guide.md` - practical steps for upgrading an
  existing Solid 1.x (or SolidStart / TanStack Start) project: dependency
  and config changes, ecosystem gotchas, and the full rename/removal table.
- `references/dev-diagnostics.md` - every dev-mode diagnostic code (error
  and warning), what triggers it, and the fix.
- `references/runtime-size-and-async-tradeoff.md` - Ryan Carniato's July 16,
  2026 explanation of why Solid 2.0 is less tree-shakeable than 1.x, why async
  support stays in the baseline runtime, the roughly 5 kB -> 10 kB minzipped
  counter-example trade-off, and the related design discussion in issue #2883.
