---
name: effect-v4-conventions
description: >-
  Use whenever writing, reviewing, fixing, or migrating TypeScript code that
  uses the Effect library (the effect npm package, including @effect/platform,
  @effect/schema, @effect/cluster, @effect/rpc, @effect/ai, @effect/sql,
  @effect/workflow, etc). MUST consult this before writing any Effect code,
  since v3 APIs (Effect.catchAll, Effect.fork, FiberRef, Context.Tag, Either,
  Runtime.runFork, Scope.extend, Schema.transform, Mailbox, etc) are renamed,
  restructured, or removed in v4 and will silently produce wrong or
  non-compiling code if used from memory. Trigger on effect, Effect.gen,
  effect-ts, migrating v3 to v4, upgrading effect, Schema/Context/Layer/Fiber/
  Cause/Scope/Runtime code in the effect ecosystem, or any error mentioning
  these effect module names.
---

# Effect v4 Conventions

Reference for writing correct Effect v4 code and migrating Effect v3 code to
v4. The `effect` package went through a large breaking-change rewrite from v3
to v4 — renamed modules, renamed APIs, restructured data types, and removed
patterns. Do not rely on v3-era memory of the Effect API; check this skill
first, since many v3 names still "look right" but are wrong or gone in v4.

## How to use this skill

1. **Writing new Effect code or reviewing/fixing existing code**: assume v4
   unless the user's `package.json` / import paths clearly pin `effect@^3`.
   Check the cheat sheet below first — it covers the renames you'll hit most
   often. If the construct isn't there, open the matching reference file.
2. **Migrating v3 → v4**: go module by module. For each v3 API in the source,
   look it up in `references/import-api-map.md` first (it's the master
   rename table), then check the topic-specific reference file for anything
   marked `semi-auto`, `manual`, or `restructure` — those need real code
   changes, not just a find-and-replace.
3. Don't guess at an unlisted API — grep the reference files for the v3 name
   before assuming a 1:1 rename exists. Some v3 APIs are just **removed**
   with no replacement (see each file's "removed" entries).

## Reference files (topics)

| File | Covers |
| --- | --- |
| `references/import-api-map.md` | Master v3→v4 import path map (290 modules) + 53 simple API renames. **Start here for any unfamiliar rename.** |
| `references/cause.md` | `Cause<E>` flattened from a tree (`Sequential`/`Parallel`) to a flat `reasons` array; `*Exception` → `*Error`. |
| `references/schema.md` | Full `Schema` v3→v4 map: `transform`, `filter`, `optionalWith`, `pick`/`omit`, `extend`, `Union`/`Tuple` variadic→array, `*FromSelf` renames, filter `is*` renames. |
| `references/services.md` | `Context.Tag` / `Context.GenericTag` / `Effect.Tag` / `Effect.Service` all unified into `Context.Service`; accessor proxies replaced by `.use`/`.useSync`. |
| `references/yieldable.md` | `Ref`, `Deferred`, `Fiber` etc. are no longer `Effect` subtypes — they're `Yieldable` only. `yield*` still works; passing them to combinators needs `.asEffect()` or module functions (`Ref.get`, `Deferred.await`, `Fiber.join`). |
| `references/equality.md` | `Equal.equals` is structural by default now (was reference equality); `NaN === NaN` is now true; `byReference` to opt out; `equivalence` → `asEquivalence`. |
| `references/error-handling.md` | `catchAll*` → `catch*`; `catchSome`/`catchSomeCause` → `catchFilter`/`catchCauseFilter` (uses `Filter` module, not `Option`); new `catchReason(s)`, `catchEager`. |
| `references/fiber-keep-alive.md` | Process no longer exits early while a fiber is suspended — keep-alive is built into core now. `runMain` still recommended for signal handling/exit codes. |
| `references/fiberref.md` | `FiberRef`/`FiberRefs`/`Differ` removed entirely → `Context.Reference` (`References.*`). `Effect.locally`/`FiberRef.set` → `Effect.provideService`. |
| `references/forking.md` | `Effect.fork` → `forkChild`, `Effect.forkDaemon` → `forkDetach`; all fork variants take `{ startImmediately?, uninterruptible? }`; `forkAll`/`forkWithErrorHandler` removed. |
| `references/generators.md` | `Effect.gen(this, function*() {...})` → `Effect.gen({ self: this }, function*() {...})`. |
| `references/layer-memoization.md` | Layers are now memoized **across** `Effect.provide` calls (was per-call in v3). Use `Layer.fresh` or `Effect.provide(layer, { local: true })` to opt out. |
| `references/runtime.md` | `Runtime<R>` type removed. `Runtime.runFork(runtime)(effect)` → `Effect.runForkWith(services)(effect)` with `Effect.context<R>()`. |
| `references/scope.md` | `Scope.extend` → `Scope.provide` (same behavior, new name, data-first and curried forms both supported). |

## Cheat sheet: most common renames

Check here before opening a reference file — this covers the highest-frequency
hits across all topics.

```text
Effect.catchAll          -> Effect.catch
Effect.catchAllCause     -> Effect.catchCause
Effect.catchSome         -> Effect.catchFilter        (Filter, not Option)
Effect.catchSomeCause    -> Effect.catchCauseFilter
Effect.fork              -> Effect.forkChild
Effect.forkDaemon        -> Effect.forkDetach
Effect.zipRight          -> Effect.andThen
Effect.zipLeft           -> Effect.tap
Effect.either            -> Effect.result
Effect.async             -> Effect.callback
Effect.gen(this, fn)     -> Effect.gen({ self: this }, fn)
Context.Tag / .GenericTag/ Effect.Tag / Effect.Service -> Context.Service
Effect.runtime<R>() + Runtime.runFork(rt) -> Effect.context<R>() + Effect.runForkWith(services)
FiberRef.*               -> Context.Reference (References.*)
Effect.locally            -> Effect.provideService
Scope.extend              -> Scope.provide
Layer.scoped               -> Layer.effect
Either / Either.right / Either.left -> Result / Result.succeed / Result.fail
Mailbox / Mailbox.make     -> Queue.Queue / Queue.make
Effect.makeSemaphore(Unsafe) -> Semaphore.make(Unsafe)
Ref / Deferred / Fiber as Effect (yield* ref directly) -> Ref.get(ref) / Deferred.await(d) / Fiber.join(f)
Schema.transform            -> schema.pipe(Schema.decodeTo(to, SchemaTransformation.transform({decode, encode})))
Schema.filter(pred)         -> schema.check(Schema.makeFilter(pred))
Schema.Union(A, B) / Tuple(A, B) / Literal("a","b") -> Union([A, B]) / Tuple([A, B]) / Literals(["a","b"])
Schema.pick("a") / omit("a") -> mapFields(Struct.pick(["a"])) / mapFields(Struct.omit(["a"]))
*FromSelf (Date, Duration, Chunk, Cause, Exit, Option, ...) -> drop the FromSelf suffix
Cause tree (Sequential/Parallel/Empty)  -> Cause.reasons: ReadonlyArray<Reason>
*Exception (NoSuchElementException, TimeoutException, ...) -> *Error
Equal.equals on plain objects/arrays: reference equality -> structural equality by default
```

## Key structural changes to keep in mind while writing code

- **Services**: always define with `Context.Service`, never `Context.Tag`,
  `Effect.Tag`, or `Effect.Service`. No auto-generated `.Default` layer —
  build layers explicitly with `Layer.effect(this, this.make)` and name them
  `layer` (not `Default`/`Live`).
- **`Ref`/`Deferred`/`Fiber` are not `Effect`s anymore.** `yield*` still works
  on them via `Yieldable`, but never pass them straight into `Effect.map`,
  `Effect.all`, etc. — use `Ref.get`, `Deferred.await`, `Fiber.join`, or
  `.asEffect()`.
- **`Either` is gone** — it's `Result` now (`Result.succeed`/`Result.fail`).
- **`Cause`** is a flat `{ reasons: Reason[] }`, not a tree — iterate, don't
  pattern-match `Sequential`/`Parallel`.
- **Layers auto-memoize across `Effect.provide` calls** — don't assume you
  need to hoist every layer into one `provide` call to avoid double-init
  (though composing before providing is still the cleaner pattern).
- **No `runMain` needed just to keep the process alive** — but still use it
  for signal handling and exit codes in real entrypoints.

## Quick sanity check before finishing a task

After writing or migrating Effect code, scan for these v3 tells that mean
something was missed: `Context.Tag(`, `Effect.Tag(`, `Effect.Service<`,
`FiberRef.`, `Runtime.runFork`, `Scope.extend`, `Effect.fork(` (bare, not
`forkChild`), `catchAll`, `catchSome(`, `Mailbox`, `Either.`, `*FromSelf`,
`Schema.filter(`, `Schema.transform(`, `Effect.gen(this,`.