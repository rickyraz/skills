# Runtime size, tree-shaking, and the async baseline

This note explains a Solid 2.0 design trade-off discussed by Ryan Carniato:
async is no longer isolated behind one opt-in primitive, so some async-related
runtime semantics are part of the baseline reactive model.

Use this note when reasoning about:

- minimum runtime size;
- tree-shaking limits;
- "sync-only" runtime ideas;
- benchmark comparisons;
- why async support exists even when an application does not explicitly import
  a dedicated async primitive.

This document distinguishes **historical July 2026 observations** from the
**current optimization direction** on the Solid 2.0 `next` branch.

---

## Design takeaway

Solid 2.0 makes async a property of the reactive graph rather than a feature
owned by one dedicated API.

A Promise or AsyncIterable can enter the graph through ordinary computations,
and the runtime is expected to handle that correctly.

That has an important consequence:

```text
there is no single import that means:
"this application uses async"
```

Unlike a model where one explicit async primitive activates all async support,
Solid 2.0 can encounter async through:

- `createMemo`;
- function-form signals/stores;
- projections;
- effects;
- lazy boundaries;
- actions;
- other graph computations.

So the runtime cannot reduce async support to a single optional
`createResource`-style addon.

However, this does **not** mean that every async-, transition-, optimistic-, or
store-related implementation path must remain permanently reachable in every
bundle.

The current issue #2883 work makes an important distinction:

```text
async semantics are baseline
!=
all async implementation is unshakable
```

---

## Historical context: Ryan Carniato, July 16, 2026

Ryan described two broad ways libraries stay small:

1. **incrementally additive packaging**
   - ship a small core;
   - install optional feature packages;

2. **tree-shakeable packaging**
   - publish one package;
   - bundlers remove unreachable feature paths.

Solid 1.x was especially strong at the second strategy. Feature-specific
imports and internal flags let bundlers prune code down to fine-grained
conditional paths.

Ryan's July 2026 observation was that Solid 2.0 became less tree-shakeable at
the minimum-runtime level because the streamlined API removed a clean feature
boundary around async.

His point was roughly:

```text
Solid 1.x:
dedicated feature boundaries
-> bundler can often prove feature is unused

Solid 2.0:
async can enter almost anywhere in the graph
-> core cannot assume "no async"
```

At that point in development, he compared a tiny signal counter at roughly:

```text
Solid 1.x
~just under 5 kB minzipped

Solid 2.0
~just under 10 kB minzipped
```

and attributed much of the difference to async-related runtime machinery that
was still retained in the baseline build.

Treat those numbers as a **historical snapshot**, not as a permanent Solid 2.0
size guarantee.

---

## Why there is no simple "async import switch"

A feature is easy to tree-shake when a bundler can prove:

```text
feature API not imported
-> feature code cannot execute
-> remove it
```

Solid 2.0 intentionally weakens that assumption for async.

There is no one API whose presence uniquely means "async mode is enabled":

- there is no `createResource` requirement;
- `Loading` is optional;
- `lazy` is optional;
- no explicit transition wrapper is required;
- an ordinary computation can return a Promise or AsyncIterable.

So this mental model is wrong:

```text
no async-specific import
-> runtime can be sync-only
```

The runtime must preserve enough graph semantics to safely encounter
async-shaped computations.

---

## Important correction: baseline async semantics do not imply a permanently monolithic async implementation

Later bundle analysis in Solid issue #2883 made the situation more nuanced.

The problem was not simply:

```text
"async exists, therefore nothing can tree-shake"
```

The audit identified specific retention mechanisms, including:

- direct imports from hot core functions;
- transition/queue logic retained through instantiated classes;
- feature hook installers living in modules that core imports eagerly;
- store/projection coupling through static imports.

That means some runtime mass can potentially be made more tree-shakeable
without removing Solid 2.0's async semantics.

The better model is:

```text
baseline semantic contract
├─ core must understand pending/error/async-shaped graph behavior
│
└─ implementation subsystems
   ├─ transitions
   ├─ optimistic state
   ├─ affects
   ├─ projections
   └─ parts of store/async helpers
      may still be separable behind better tree-shaking seams
```

So do not equate:

```text
"async is universal"
```

with:

```text
"every byte related to async must always ship"
```

---

## Current issue #2883 direction

A later audit in issue #2883 measured the current retention graph in more
detail.

One reported measurement using Rollup + terser put a minimal core fixture at
roughly:

```text
18.8 kB minified
7.5 kB gzip
```

while the same general floor was larger with esbuild in that experiment.

The audit also estimated that with hook-based subsystem separation, a
signal-focused core could theoretically move toward approximately:

```text
3.9–4.2 kB gzip
```

for the signals/core floor, or around:

```text
~5.5 kB gzip
```

for a minimal app including the web runtime, depending on the final
implementation and bundler.

These are **engineering measurements and optimization targets**, not API
guarantees.

The important architectural lesson is not the exact number. It is:

```text
the runtime floor is still being optimized
without reverting the async-first semantic model
```

---

## What "less tree-shakeable" should mean in this note

Use this phrasing:

```text
Solid 2.0 has fewer clean feature boundaries around async than Solid 1.x,
because async can enter ordinary reactive computations.

That makes some async-related runtime semantics load-bearing in the baseline.

Implementation coupling can still be improved so unused subsystems shake out
more effectively.
```

Avoid the stronger statement:

```text
"async machinery cannot be tree-shaken"
```

because current work shows that substantial parts of the implementation can be
made more tree-shakeable even though async capability remains part of the core
contract.

---

## Why a separate `sync-only` runtime is still not the normal recommendation

There is an important difference between:

```text
making unused subsystems tree-shakeable
```

and:

```text
publishing a separate sync-only Solid runtime
```

The first preserves one ecosystem and one semantic contract.

The second creates a runtime with different capability guarantees.

A sync-only build could be smaller in a benchmark, but code or libraries that
legitimately return async values into the graph would no longer compose with it
under the same assumptions.

So do not recommend a hypothetical:

```text
solidjs/sync-only
```

as the normal solution to bundle-size concerns.

The useful optimization target is:

```text
one Solid runtime contract
+
better tree-shaking of optional implementation subsystems
```

not:

```text
two incompatible runtime capability tiers
```

---

## Bundler choice matters

Do not quote one runtime-size number without naming the measurement method.

The issue #2883 audit found meaningful differences between bundlers and build
paths.

For example, the same kind of minimal fixture could retain different amounts
under:

- Rollup + terser;
- esbuild;
- source builds;
- published flat distribution builds.

Therefore:

```text
"Solid 2.0 is X kB"
```

is usually too imprecise.

Prefer:

```text
"Under bundler/configuration Y, fixture Z measured approximately X kB gzip."
```

---

## Tree-shaking vs code splitting

Tree-shaking and code splitting solve different problems.

### Tree-shaking

Removes code that the bundler can prove is unreachable.

```text
unused implementation
-> removed from final chunk
```

### Code splitting

Moves reachable code into later-loaded chunks.

```text
needed eventually
-> not required in initial chunk
```

Route splitting, dynamic imports, and multi-file publishing can improve initial
load behavior even when a capability cannot be eliminated entirely.

But code splitting does not automatically solve a hot-core retention problem if
the core still statically references the implementation.

Issue #2883 specifically found that simply splitting the distribution into
chunks would recover less than fixing the underlying dependency seams.

---

## Capability-for-floor-size trade-off

The original Ryan statement is still useful as a design explanation:

Solid 2.0 chooses to make the baseline reactive model understand richer async
behavior.

That can increase the minimum amount of runtime logic compared with a system
whose tiny baseline is intentionally sync-only.

The trade-off should now be stated as:

```text
Solid 2.0 accepts a richer baseline semantic contract
in exchange for simpler async composition throughout the graph.

The implementation is still expected to aggressively tree-shake
subsystems that are not required to preserve that contract.
```

That is more precise than saying the extra runtime size is simply unavoidable.

---

## Minimum runtime size vs runtime capability

When comparing Solid 1.x and 2.0, distinguish these two questions.

### Minimum runtime size

```text
How small can a particular production fixture bundle?
```

This depends on:

- framework version;
- bundler;
- minifier;
- tree-shaking implementation;
- renderer;
- imported APIs;
- package layout.

### Runtime capability included by default

```text
What behavior is the core runtime prepared to handle?
```

Solid 2.0's baseline capability includes async-aware graph semantics more
deeply than Solid 1.x.

These are related, but they are not the same metric.

---

## How to talk about the July ~5 kB -> ~10 kB comparison

Acceptable:

```text
During July 2026 development, Ryan described a tiny counter moving from
roughly sub-5 kB to roughly sub-10 kB minzipped, with much of the added floor
coming from async-related machinery retained at that point.
```

Do not write:

```text
Solid 2.0 has a fixed 10 kB minimum.
```

Also do not write:

```text
Solid 2.0 cannot tree-shake async code.
```

The current issue work has already made both statements too strong.

---

## Practical guidance

When bundle size matters:

1. **Measure the real production application.**

   Do not extrapolate solely from a toy counter.

2. **Still measure a minimum fixture when investigating runtime floor.**

   Toy fixtures are useful diagnostics for framework/runtime overhead; they
   simply are not the whole application-performance story.

3. **Record the bundler and minifier.**

   A size number without build configuration is incomplete.

4. **Treat historical numbers as snapshots.**

   The July 2026 ~5 kB / ~10 kB comparison explains the design discussion, not
   a permanent RC/stable floor.

5. **Distinguish semantic baseline from implementation retention.**

   Async-aware graph semantics may be load-bearing while transitions,
   optimistic state, projections, affects, or other machinery may still become
   more tree-shakeable.

6. **Continue normal ESM tree-shaking.**

   Application and ecosystem code should still be structured so unused
   exports/modules can disappear.

7. **Use route/code splitting for loading behavior.**

   Even code that cannot be eliminated can sometimes be delayed until a route
   or feature needs it.

8. **Do not recommend a separate sync-only runtime by default.**

   Prefer the normal ecosystem-compatible runtime plus ongoing tree-shaking
   improvements.

9. **Compare capability as well as byte floor.**

   A slightly larger minimum runtime that absorbs functionality otherwise
   requiring separate systems can be a reasonable trade-off.

10. **Re-check issue #2883 before quoting current numbers.**

    This area is actively changing during the Solid 2.0 RC cycle.

---

## Guidance for coding agents

When answering a bundle-size question:

### Do

Say:

```text
Solid 2.0's async-first graph makes some async semantics part of the core
runtime, so its minimum floor historically increased relative to Solid 1.x.
However, the team is actively making implementation subsystems more
tree-shakeable, so old ~10 kB figures should not be treated as a fixed
minimum.
```

### Do not

Say:

```text
Solid 2.0 always ships 5 kB of unavoidable async code.
```

Do not recommend:

```text
use a sync-only Solid build
```

unless such a supported runtime actually exists in current upstream releases.

Do not compare framework bundle sizes without specifying:

```text
version
fixture
bundler
minifier
compression
renderer
```

---

## Historical source statement

The original July 16, 2026 Ryan Carniato statement remains useful context.

Its core argument was:

- Solid 1.x benefited heavily from fine-grained tree-shaking;
- Solid 2.0 removed a clean opt-in boundary around async;
- async values may appear anywhere in the graph;
- a separate sync-only runtime would be undesirable for ecosystem composition;
- this raised the runtime floor in the implementation available at that time.

Preserve the quote as historical evidence if needed, but keep current
measurements and conclusions outside the quote.

Do not paraphrase the July numbers as current release guarantees.

---

## Related upstream discussion

Primary design/performance thread:

- Solid issue **#2883**

Current takeaway from that thread:

```text
initial framing:
async-first semantics increased the minimum retained runtime significantly

later audit:
the semantic point remains,
but several implementation-retention paths can be refactored behind
tree-shakeable seams
```

Re-check the issue and current `next` branch before updating benchmark numbers.

---

## Rule of thumb

Use this compact mental model:

```text
Solid 1.x
-> exceptional async feature paths
-> extremely strong minimum tree-shaking

Solid 2.0
-> async-aware reactive graph by default
-> richer baseline semantics
-> higher coupling risk in the minimum runtime
-> active work to recover tree-shaking through better subsystem seams
```

Or even shorter:

```text
async semantics are baseline;
async implementation does not all have to be.
```
