# React View Transitions Architecture Reference

Load this reference when you need React implementation details for View Transitions: commit phase sequencing, pairing, auto-naming, callbacks, gesture clones, resource blocking, animation optimization, lane gates, or known bugs.

## Availability And Version Gate

As of April 2026, React's official docs mark `<ViewTransition />` as available only in the Canary and Experimental channels. Do not assume stable React 19.x exposes `ViewTransition` or `addTransitionType`.

Before applying the internals in this reference to an app, validate:

- `react` and `react-dom` are both Canary, or both Experimental.
- The package channels match; do not mix stable `react` with Canary `react-dom`, or the reverse.
- Framework-level APIs such as router view-transition options do not remove the need for compatible React packages when using React's built-in component API.

For stable React apps, use the browser `document.startViewTransition()` API directly or recommend upgrading both React packages to Canary/Experimental if the user wants React's `<ViewTransition />`.

## File Map

| File | Role |
|------|------|
| `ReactFiberCommitViewTransitions.js` | Core orchestration: enter, exit, update, pairing |
| `ReactFiberViewTransitionComponent.js` | `ViewTransitionState`, auto-naming, `className` resolution |
| `ReactFiberConfigDOM.js` | DOM/browser API: `startViewTransition`, font/image blocking, animation optimization |
| `ReactFiberApplyGesture.js` | Gesture clone architecture |
| `ReactFiberWorkLoop.js` | Lane gate, `shouldStartViewTransition` flag |

## Commit Phase Integration

`startViewTransition()` wraps only the mutation phase. Everything before is old-state setup. Everything after is new-state setup.

```text
[Before Mutation]
  commitExitViewTransitions()        -> assign view-transition-name to OLD DOM
  appearingViewTransitions map       -> populated with entering Fiber names for pairing

[Mutation - inside browser startViewTransition.update()]
  commitBeforeUpdateViewTransition() x2
    1st: names on current (old) tree
    2nd: names on finishedWork (new) tree

[After Mutation]
  commitEnterViewTransitions()       -> assign names to NEW DOM
  measureViewTransitionHostInstances() -> detect if layout actually changed
  restoreUpdateViewTransition()      -> cancel if nothing moved

[Passive]
  restoreNestedViewTransitions()     -> cleanup nested VT whose parent mutated but they did not
```

## ViewTransitionState

```ts
type ViewTransitionState = {
  autoName: null | string,            // lazy, base-32 encoded, assigned at commit not render
  paired: null | ViewTransitionState, // temp cross-ref during commit, cleared after
  clones: null | Array<Instance>,     // gesture phase DOM clones
  ref: null | ViewTransitionInstance,
};
```

Auto-name format:

```text
_${prefix}t_${globalClientIdCounter.toString(32)}_
```

If `props.name` is set and is not `"auto"`, the explicit name wins.

## Pairing System

`appearingViewTransitions: Map<string, ViewTransitionState>` is populated before mutation with entering View Transitions.

During `commitExitViewTransitions()`:

```text
name -> lookup in appearingViewTransitions
  Found?     -> SHARE: both sides marked paired, onShare fires, deleted from map
  Not found? -> EXIT: onExit fires
```

| Scenario | Callback |
|----------|----------|
| Appears, no pair | `onEnter` |
| Disappears, no pair | `onExit` |
| Both sides matched by name | `onShare` |
| Gesture | `onGestureEnter` |

Edge case: exiting element outside viewport -> names reverted, no animation. Entering element outside viewport after pairing -> `onShare` still fires. This is a known bug.

## `className === "none"` Early Exit

Every transition function calls `getViewTransitionClassName()` first:

```js
if (className === "none") return; // skip; no view-transition-name assigned
```

Known bug: `update="foo"` + `layout="none"` can still apply `"foo"` incorrectly for a layout-only change. The `Update` Fiber flag is a proxy and does not distinguish content changes from layout-only changes.

## Resource Blocking

Resource blocking happens inside `startViewTransition.update()`.

```text
1. Font check: were fonts.status "loaded" before mutation?
   -> forceLayout()
   -> if now "loading", push document.fonts.ready to blockingPromises
   This ignores fonts already loading before commit to avoid unrelated blocks.

2. Viewport images only:
   -> estimate bytes cumulatively
   -> if total > estimatedBytesWithinLimit, abandon image blocking
   -> else push load promise

3. Race:
   blockingPromises vs setTimeout(500ms)
   winner calls layoutCallback()
```

## Animation Optimization - Keyframe Pruning

After browser keyframe generation, React removes identical `width` and `height` keyframes:

```text
for each ::view-transition animation:
  if width is identical across all keyframes:
    -> remove width
    -> let CSS compute it
    -> verify computed style matches
    -> if mismatch, restore width

  repeat for height
```

This avoids GPU compositing work when only position changed, not size.

## Gesture Support

Gestures clone the entire root container, not individual elements:

```text
detectMutationOrInsertClones(finishedWork)
  mutations found -> cloneRootViewTransitionContainer()
                     + recursivelyInsertClones()
                     + root.gestureClone = rootClone
  no mutations    -> cancelRootViewTransitionName()
                     + root.gestureClone = null
```

Clones are stored in `ViewTransitionState.clones[]`.

## Lane Gate

```js
const eligible =
  enableViewTransition &&
  includesOnlyViewTransitionEligibleLanes(committedLanes);
```

Sync urgent lanes and retry lanes are ineligible. This prevents transitions on keyboard input.

## Reordering Limitation

```text
Wrong: ViewTransition tracks individual children inside its boundary
Right: ViewTransition treats its entire subtree as one opaque layer
```

Multiple `HostInstance` children get names assigned in DOM order at snapshot time. DOM order changes produce cross-fades, not position swaps.

Fix: each list item needs its own stable name, for example `<ViewTransition name={"item-" + id}>`.

## Async / Suspense Constraint

`document.startViewTransition()` is atomic. React cannot:

- Feed streaming chunks as intermediate states
- Animate between partially hydrated states
- Chain multiple Suspense reveals in one transition

Workaround: `accumulateSuspenseyCommit` + `suspendOnActiveViewTransition` holds commit until the previous transition completes.

## Why Userland Cannot Fully Replicate It

GSAP and Framer FLIP operate on DOM transforms only. Native view transitions operate below the DOM at the compositing layer.

| Capability | Userland | Native |
|-----------|----------|--------|
| Capture composited output, such as Canvas or WebGL | No | Yes |
| Suspend rendering pipeline mid-frame | No | Yes |
| `::view-transition-*` pseudo-elements | No | Yes |
| Font/image decode pipeline coordination | Approximate | Yes |

## Key Constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `SUSPENSEY_FONT_AND_IMAGE_TIMEOUT` | 500ms | Max wait before proceeding without fonts/images |
| `estimatedBytesWithinLimit` | threshold | Image byte budget |
| `globalClientIdCounter` | base-32 increments | Auto-name seed |
| `__reactViewTransition` | doc property | Current running transition handle |
