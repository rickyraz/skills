# SolidJS 2.0 anti-patterns

Recurring mistakes models make when writing Solid 2.0 code, each caused by a
React or Solid-1.x reflex firing instead of the 2.0 idiom. Each entry gives
the wrong version, the fix, and *why* the wrong version feels natural - the
reasoning is what lets the correction generalize instead of only fixing one
line.

**Contents:** [1. Accessor passed as prop](#1-passing-an-accessor-as-a-prop-instead-of-calling-it-at-the-call-site) ·
[2. Manual class strings](#2-manual-class-string-building-instead-of-class--) ·
[3. Controlled inputs](#3-uncontrolled-inputs-written-as-controlled) ·
[4. Flat hook returns](#4-flat-object-return-from-a-custom-hook-instead-of-state-actions) ·
[5. Mixed accessor/property reads](#5-mixing-accessor-calls-and-property-reads-in-one-state-object) ·
[6. Parallel stores](#6-several-parallel-stores-instead-of-one-derived-pipeline) ·
[7. Wide Loading boundary](#7-loading-boundary-scoped-around-a-whole-shell-instead-of-the-suspending-part) ·
[8. Per-item mutation loops](#8-looping-one-mutation-per-item-instead-of-a-bulk-endpoint) ·
[9. Imperative splice](#9-imperative-splice-loops-instead-of-a-returned-filter) ·
[10. Premature Context](#10-reaching-for-context-plus-a-throw-wrapper-hook-before-its-earned) ·
[11. PascalCase callbacks](#11-pascalcasing-a-render-callback-like-its-a-component) ·
[12. Restated defaults](#12-restating-an-option-that-already-matches-the-default) ·
[13. Module-scope primitives](#13-reactive-primitives-created-at-module-scope) ·
[14. onCleanup vs onSettled](#14-oncleanup-for-ordinary-setupteardown-instead-of-onsettled)

## 1. Passing an accessor as a prop instead of calling it at the call site

```tsx
// Bug: the function itself is handed down as `filter`.
function ArticleList(props: { filter: () => string }) {
  return <span>{props.filter()}</span>; // double indirection
}
<ArticleList filter={activeFilter} />;

// Fix: call the accessor at the JSX boundary; the child receives a value.
function ArticleList(props: { filter: string }) {
  return <span>{props.filter}</span>;
}
<ArticleList filter={activeFilter()} />;
```

**Why:** props are reactive *values* read through a proxy - `props.filter`
already re-reads on every access, the same way `filter()` would. Passing the
function instead of its value is the single most common Solid bug pattern,
and it's not new to 2.0: it has been wrong since 1.x. It resurfaces
constantly because "pass a function so the child can read the latest value
lazily" is exactly the right instinct in a callback-based system, and Solid's
props already give you that for free without a function wrapper.

## 2. Manual class string building instead of `class={[...]}`

```tsx
// Bug
const className = () => {
  const parts = ["card"];
  if (isActive()) parts.push("active");
  if (isDisabled()) parts.push("disabled");
  return parts.join(" ");
};
<div class={className()} />;

// Fix
<div class={["card", { active: isActive(), disabled: isDisabled() }]} />;
```

**Why:** the `classnames`/template-literal habit from React is deeply
trained. `class` in Solid 2.0 accepts array and object forms directly -
array entries are always-on, object entries toggle by truthiness - so there
is never a reason to hand-build the string.

## 3. Uncontrolled inputs written as controlled

```tsx
// Bug: signal + value + onInput, purely to read the value once on submit.
const [text, setText] = createSignal("");
<input value={text()} onInput={(e) => setText(e.currentTarget.value)}
  onKeyDown={(e) => e.key === "Enter" && submit(text())} />;

// Fix: read the DOM directly when you need it; no signal required.
<input onKeyDown={(e) => {
  if (e.key !== "Enter") return;
  submit(e.currentTarget.value);
  e.currentTarget.value = "";
}} />;
```

**Why:** React makes uncontrolled inputs awkward, so controlled `value` +
`onChange` pairs dominate training data and get reached for reflexively.
Solid's JSX doesn't reconcile DOM state against a virtual tree, so the DOM
element can simply hold its own value - only wrap it in a signal when
something *outside* the input needs to read or clear it.

## 4. Flat object return from a custom hook instead of `[state, actions]`

```ts
// Bug: mirrors React's `useThing()` object-return convention.
function useCart() {
  return { get items() { return data.items; }, addItem, removeItem };
}
const cart = useCart();
cart.items; // reactive read
cart.addItem(x); // action

// Fix: mirror Solid's own primitive shape - a tuple of (state, actions).
function useCart() {
  const state = { get items() { return data.items; } };
  const actions = { addItem, removeItem };
  return [state, actions] as const;
}
const [cart, { addItem, removeItem }] = useCart();
```

**Why:** `createSignal` returns `[read, write]`, `createStore` returns
`[store, setter]` - Solid's whole convention is "tuple of (read, write)" so
custom hooks composing several reads and writers should mirror it. The
tuple shape also makes the destructuring rule visible at the call site:
never destructure `state` (reads must stay bound to the proxy/getter),
always safe to destructure `actions` (stable callbacks).

## 5. Mixing accessor calls and property reads in one state object

```ts
// Bug: consumer has to remember which key is call-this vs read-this.
const state = {
  get items() { return data.items; }, // property read
  status, // signal accessor - must be called: state.status()
};

// Fix: expose every reactive read as a property read.
const state = {
  get items() { return data.items; },
  get status() { return status(); },
};
```

**Why:** a signal's "call it to read it" convention is correct *inside* the
module that owns the signal, but leaks as an inconsistent API once it's
bundled into a hook's return surface alongside store properties. Wrapping
the signal in a getter gives every consumer one uniform read shape.

## 6. Several parallel stores instead of one derived pipeline

```ts
// Bug: three independently-written sources kept in sync by hand.
const [data] = createOptimisticStore(() => api.listItems(), []);
const [meta, setMeta] = createStore<Record<string, { saving?: boolean; error?: string }>>({});
// every action now has to write to `data` *and* `meta`, and keep them aligned

// Fix: one derived source; side channels are plain (non-reactive) objects
// applied inside the projection function, not stored as reactive state.
const errors: Record<string, string> = {}; // plain JS, not a signal/store

const [items, setItems] = createOptimisticStore<Item[]>(
  async () => {
    const rows = await api.listItems();
    applyErrors(rows, errors); // paint error state onto rows here
    return rows;
  },
  [],
  { key: "id" }
);

const save = action(function* (item: Item) {
  setItems((list) => { /* optimistic write */ });
  try {
    yield api.save(item);
    delete errors[item.id];
  } catch {
    errors[item.id] = "save failed";
  }
  refresh(items); // re-runs the projection, which re-applies `errors`
});
```

**Why:** this is the highest-impact pattern in this list. Redux/React
training data is dominated by reducers that write each kind of state (data,
loading, error) into its own flat slice. Solid 2.0's derived-store form
exists specifically so you can compose "server data + side-channel
overlays" inside one projection function - the side channel doesn't need to
be reactive at all, since mutating it and calling `refresh()` is the trigger.
Reaching for "N pieces of state, N stores" instead of one pipeline is the
default failure mode; treat one derived store with a composing projection as
the starting design, not an optimization.

## 7. `Loading` boundary scoped around a whole shell instead of the suspending part

```tsx
// Bug: fallback re-states the same chrome that the resolved content has,
// so a revalidation tears down and rebuilds the header, nav, everything.
<Loading fallback={<AppShell><p>Loading…</p></AppShell>}>
  <AppShell><Dashboard /></AppShell>
</Loading>

// Fix: only the part that actually reads async data sits inside the boundary.
<AppShell>
  <Loading fallback={<p>Loading…</p>}>
    <Dashboard />
  </Loading>
</AppShell>
```

**Why:** React suspense boundaries are cheap to wrap widely because the
reconciler diffs virtual nodes - the "duplication" between fallback and
content is amortized away. Solid's JSX builds real DOM: the fallback subtree
is actually constructed, and actually torn down and replaced on transition.
There's no diff to hide the cost, so the right reflex is the opposite of
React's: scope `Loading` as tightly as possible around the suspending read,
and keep everything else outside so it mounts once and stays put.

## 8. Looping one mutation per item instead of a bulk endpoint

```ts
// Bug: N network round-trips disguised as one action.
const archiveAll = action(function* (ids: string[]) {
  for (const id of ids) yield api.archive(id);
  refresh(items);
});

// Fix: one round trip; fan a bulk failure back out to per-item state if needed.
const archiveAll = action(function* (ids: string[]) {
  try {
    yield api.archiveMany(ids);
  } catch {
    ids.forEach((id) => (errors[id] = "archive failed"));
  }
  refresh(items);
});
```

**Why:** it's easy to extend a working per-item action with a loop because
the shape is already there, but a real backend rarely exposes an N-request
bulk operation and the example code shouldn't pretend it does. One `yield`
per network round trip is the rule of thumb.

## 9. Imperative splice loops instead of a returned `filter`

```ts
// Bug: index-hunting and splicing inside the draft.
setItems((draft) => {
  const i = draft.findIndex((x) => x.id === targetId);
  if (i >= 0) draft.splice(i, 1);
});

// Fix: return a filtered array from the setter.
setItems((draft) => draft.filter((item) => item.id !== targetId));
```

**Why:** a draft setter can mutate *or* return a value for shallow
replacement - for arrays, the return path replaces by index and adjusts
length, so surviving object references keep their identity. That makes
`filter`/`map`-style returns both correct and far less error-prone than
manual index arithmetic. This is positional replacement, not keyed
reconciliation - the keyed diffing behavior belongs to the projection-function
form (`createStore(fn, seed, { key })`), not to a plain setter's return path.

## 10. Reaching for Context (plus a throw-wrapper hook) before it's earned

```ts
// Bug: two hops of prop drilling "solved" with Context and a React-style
// non-null wrapper hook.
const CartContext = createContext<CartApi>();
const useCart = () => {
  const ctx = useContext(CartContext);
  if (!ctx) throw new Error("missing CartContext");
  return ctx;
};

// Fix: for shallow drilling, just pass props.
function App() {
  const cart = createCart();
  return <Header cart={cart} />;
}
```

**Why:** "shared state means Context" is a strong React reflex, but Solid's
Context has a real cost and prop drilling two or three levels is usually
more direct to read. When a context genuinely does have several fanned-out
consumers at mixed depth, reach for Context - but a default-less
`createContext<T>()` already throws on a missing provider at runtime (see
`references/api-cheatsheet.md#context`), so the manual wrapper-hook pattern
should be deleted outright rather than reproduced: call `useContext`
directly. Never substitute a module-level singleton for Context as a way to
avoid prop drilling - it has no disposal and leaks across requests under SSR.

## 11. PascalCasing a render callback like it's a component

```tsx
// Bug: named and cased like a component, but it isn't mountable as one -
// `Errored` calls it directly with (err, reset), not with a props object.
function ErrorFallback(err: unknown, reset: () => void) {
  return <button onClick={reset}>Retry: {String(err)}</button>;
}
<Errored fallback={ErrorFallback}>{/* ... */}</Errored>;

// Fix: camelCase signals "callback, not component".
const renderError = (err: unknown, reset: () => void) => (
  <button onClick={reset}>Retry: {String(err)}</button>
);
<Errored fallback={renderError}>{/* ... */}</Errored>;
```

**Why:** in React, anything PascalCase returning JSX is a Component by
convention. Solid's JSX hosts two incompatible call shapes on the same
surface: actual components (`(props) => Element`, mounted via `<X />`) and
render callbacks passed to control-flow primitives (`Show`, `For`,
`Errored`, `Repeat`, `Switch`/`Match`), which the parent invokes directly
with positional arguments. PascalCasing a callback misleads the reader into
thinking it's mountable as `<X />`, which it isn't.

## 12. Restating an option that already matches the default

```ts
// Bug: adds noise, and implies `key: "id"` is a choice the caller must make.
const [items] = createOptimisticStore(() => api.list(), [], { key: "id" });

// Fix: omit it - "id" is already the default identity field.
const [items] = createOptimisticStore(() => api.list(), []);
```

**Why:** writing every option "to be safe" is a reasonable defensive habit
in general, but in Solid's projection-based primitives the default
(`options.key === "id"`) is a deliberate convention, not an implementation
detail to hedge against. Showing the call with no options communicates "this
data has an `id`, so it just works"; restating the default communicates the
opposite. Only pass `key` when the identity field genuinely isn't `id`.

## 13. Reactive primitives created at module scope

```ts
// Bug: works, but the signal and its listener live for the process lifetime.
export const [theme, setTheme] = createSignal(getInitialTheme());
window.addEventListener("storage", () => setTheme(getInitialTheme()));

// Fix: wrap it in a custom primitive owned by the component that uses it.
export function createThemeSignal() {
  const [theme, setTheme] = createSignal(getInitialTheme());
  onSettled(() => {
    const onStorage = () => setTheme(getInitialTheme());
    window.addEventListener("storage", onStorage);
    return () => window.removeEventListener("storage", onStorage);
  });
  return theme;
}
```

**Why:** `window.addEventListener` at module scope is ordinary JS muscle
memory, and "app-wide state lives in a module singleton" is an ordinary
React reflex too - nothing in Solid's API stops you from calling
`createSignal` at the top of a file. But a module-scope signal has no owner:
under SSR it becomes one instance shared by every concurrent request, and
any attached listener never gets cleaned up. Put reactive primitives (and
anything that needs cleanup) inside a component or a custom primitive
function that something else calls with an owner in scope.

## 14. `onCleanup` for ordinary setup/teardown instead of `onSettled`

```ts
// Bug: works, but is the 1.x idiom for something 2.0 has a direct primitive for.
function createResizeSignal() {
  const [size, setSize] = createSignal(measure());
  const onResize = () => setSize(measure());
  window.addEventListener("resize", onResize);
  onCleanup(() => window.removeEventListener("resize", onResize));
  return size;
}

// Fix: co-locate setup and teardown inside onSettled's returned cleanup.
function createResizeSignal() {
  const [size, setSize] = createSignal(measure());
  onSettled(() => {
    const onResize = () => setSize(measure());
    window.addEventListener("resize", onResize);
    return () => window.removeEventListener("resize", onResize);
  });
  return size;
}
```

**Why:** `useEffect(fn, [])` returning a cleanup is one of the most heavily
trained shapes for "on mount, do X, undo X on unmount" - `onSettled` is the
direct 2.0 analogue and is meant to be reached for by default, with
`onCleanup` demoted to an advanced primitive for reactive cleanup inside a
custom computation. Note also that `onCleanup` cannot even be called from
inside `onSettled` (it throws in dev) - the returned-function form is not
optional stylistic preference, it's the only supported shape there.

---

**A closing note on framing, not just code:** when asked to describe or
review Solid 2.0 code rather than write it, resist the instinct to produce a
syntax diff against React or Solid 1.x ("uses `class` not `className`, no
`key` prop, signals are called as functions…"). That inventory is true but
shallow. Ask instead what the code's structure accomplishes - e.g. how few
moving parts a layered-projection store needs compared to hand-synchronized
state - since that's usually the point of showing 2.0 code in the first
place, not the surface syntax.
