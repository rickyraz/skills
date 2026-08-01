# Migrating a Solid 1.x project to 2.0 (beta)

Practical steps for upgrading an existing app, on top of the behavioral
changes in `references/api-cheatsheet.md`. Solid 2.0 is still in beta - the
beta line is published to npm under the `next` tag, and package names /
identifiers can still shift before stable.

**Version snapshot (2026-08-01):** the `next` branch in `solidjs/solid` reports
`solid-js` version `2.0.0-beta.29` in `packages/solid/package.json`, with
`@solidjs/signals` at `^2.0.0-beta.29`. Prefer the npm `next` tag when you want
the latest compatible beta set; pin `2.0.0-beta.29` only when reproducibility
against that exact snapshot matters. The initial `2.0.0-beta.0` is not required
for experimentation or migration.

## 1. Install the beta

```bash
pnpm add solid-js@next @solidjs/web@next vite-plugin-solid@next
```

Later beta bumps: `pnpm update solid-js @solidjs/web vite-plugin-solid`. If a
fix you need only landed in `@solidjs/signals` and hasn't propagated to a
published `solid-js@next` yet, pin it with a temporary override rather than
waiting:

```json
{
  "pnpm": {
    "overrides": {
      "@solidjs/signals": "0.13.7"
    }
  }
}
```

## 2. Dedupe Solid across the workspace

If any package is linked locally (monorepo, `pnpm link`, a patched fork),
make sure only one copy of each Solid package loads - two copies of the
reactive runtime produces confusing hydration errors or silent breakage
rather than a clear error:

```ts
// vite.config.ts
export default defineConfig({
  resolve: {
    dedupe: [
      "solid-js",
      "@solidjs/web",
      "@solidjs/router",
      "@solidjs/signals",
      "@solidjs/meta",
    ],
  },
});
```

## 3. Update `tsconfig.json`

```json
{
  "compilerOptions": {
    "jsx": "preserve",
    "jsxImportSource": "@solidjs/web"
  }
}
```

Replace any `import type { JSX, ComponentProps } from "solid-js"` with the
same import from `@solidjs/web`.

## 4. Fix import paths

Run a search for these subpath imports and update them - see
`references/api-cheatsheet.md#imports-and-typescript-config` for the full
list. This is a straightforward find-and-replace; the behavioral changes
below are not.

| Old                    | New                 |
| ----------------------- | -------------------- |
| `solid-js/web`          | `@solidjs/web`       |
| `solid-js/store`        | `solid-js`           |
| `solid-js/h`             | `@solidjs/h`         |
| `solid-js/html`          | `@solidjs/html`      |
| `solid-js/universal`     | `@solidjs/universal` |

## 5. Work through behavioral changes file by file

Don't treat the rest of the migration as a rename sweep - several APIs split
into multiple replacements depending on what the old code was doing
(`createComputed` is the clearest example: it becomes `createMemo`, a split
`createEffect`, or `createSignal(fn)` depending on the call site). Go through
`references/api-cheatsheet.md` topic by topic as you touch each file, and
check `references/anti-patterns.md` before writing the replacement, not just
after something looks wrong.

Rough priority order, since later changes depend on the earlier ones:

1. Effects (`createEffect` split, `onMount` → `onSettled`, `on` helper
   removed)
2. Async (`createResource` → async computations + `Loading`,
   `startTransition`/`useTransition` removed)
3. Stores (`produce` is now default, `createMutable` → `createStore`,
   `unwrap` → `snapshot`)
4. Control flow (`Index` → `For keyed={false}`, `SuspenseList` → `Reveal`)
5. DOM (`use:` → `ref` factories, `classList` → `class`, namespace
   attributes removed)
6. Context (`Context.Provider` → context-as-provider, default-less contexts
   now throw)

## Full rename / removal map

**Renames:**

| 1.x                              | 2.0                                          |
| ---------------------------------- | ----------------------------------------------- |
| `Suspense`                         | `Loading`                                       |
| `SuspenseList`                     | `Reveal`                                        |
| `ErrorBoundary`                    | `Errored`                                       |
| `mergeProps`                       | `merge`                                         |
| `splitProps`                       | `omit`                                          |
| `createSelector`                   | `createProjection` / `createStore(fn)`          |
| `createDynamic(source, props)`     | `dynamic(source)` factory (`Dynamic` unchanged) |
| `unwrap`                           | `snapshot`                                      |
| `onMount`                          | `onSettled`                                     |
| `equalFn`                          | `isEqual`                                       |
| `getListener`                      | `getObserver`                                   |
| `classList`                        | folded into `class` (object/array forms)        |

**Removals (no 1:1 rename - see the cheatsheet for the replacement pattern):**

`createResource`, `startTransition`/`useTransition`, `batch`,
`createComputed`, the `on` helper, `onError`/`catchError`, `produce`,
`createMutable`/`modifyMutable`, `from`/`observable`, `createDeferred`,
`indexArray` (use `mapArray` with `keyed: false`), `resetErrorBoundaries`
(no longer needed - boundaries heal automatically), `enableScheduling`,
`writeSignal`, `use:` directives, `attr:`/`bool:` namespaces,
`on:`/`oncapture:`, `Context.Provider`, `solid-js/jsx-runtime` and
`solid-js/jsx-dev-runtime`.

## Ecosystem gotchas while the beta settles

**Testing library setups sometimes need extra Vite config** until a
dependency has fully moved over. A combination like this has been reported
as a working baseline for `@solidjs/testing-library` under Vitest - treat it
as a starting point to adapt, not a guaranteed fix:

```ts
// vite.config.ts
export default defineConfig({
  plugins: [solid({ ssr: true, hot: !process.env.VITEST })],
  resolve: {
    alias: {
      "solid-js/web": "@solidjs/web",
      "solid-js/store": "solid-js",
    },
  },
  environments: { ssr: { resolve: { noExternal: ["@solidjs/web"] } } },
  optimizeDeps: { include: ["@solidjs/testing-library"] },
  server: { deps: { inline: ["@solidjs/testing-library"] } },
});
```

**`flush()` shows up mostly in tests.** If a unit test sets a signal and
immediately asserts on a read, it may see the pre-write value because writes
are batched - add `flush()` (or await a microtask) before the assertion.
End-to-end tests usually don't need this since they naturally wait on
rendered output.

**Track before you `await`, not after.** This was always true, but async
`createMemo` makes it easy to write it backwards - a read after an `await`
boundary doesn't track:

```ts
// Wrong: `id()` is read after the await, so changes to `id` won't retrigger this.
const user = createMemo(async () => {
  await delay(200);
  return fetchUser(id());
});

// Right: read the tracked value first, await second.
const user = createMemo(async () => {
  const currentId = id();
  await delay(200);
  return fetchUser(currentId);
});
```

**Pay attention to dev warnings during migration**, don't just chase type
errors. `STRICT_READ_UNTRACKED` and `SIGNAL_WRITE_IN_OWNED_SCOPE` in
particular tend to catch real behavioral bugs left over from 1.x patterns
that still compile fine. See `references/dev-diagnostics.md`.

**AI-assisted migration works well but needs feedback loops.** If the
project has an existing test suite, lean on it heavily during a migration -
without one, code-generation tools tend to go in circles or introduce new
regressions while "fixing" old ones. If practical, keep a local checkout of
the Solid source (and `@solidjs/signals`) available for reference when
tracking down a signature change that isn't obvious from the compiled types
alone.
