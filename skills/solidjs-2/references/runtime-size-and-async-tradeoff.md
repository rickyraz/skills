# Runtime size, tree-shaking, and async trade-off

This note records Ryan Carniato's explanation of an intentional Solid 2.0
trade-off: a more streamlined, universally async-capable reactive model is
less tree-shakeable than Solid 1.x. Use this when reasoning about bundle size,
"sync-only" variants, benchmark comparisons, or why async support is present
even in applications that do not explicitly import an async primitive.

## Design takeaway

- Libraries can stay small by being incrementally additive (install features
  separately) or by making features highly tree-shakeable.
- Solid 1.x was especially strong at tree-shaking. Feature-specific imports and
  writable global flags let bundlers prune code paths down to conditional-block
  granularity.
- In Solid 2.0, no single API opt-in tells the runtime whether async exists.
  There is no `createResource`/`Async`/`use` boundary that uniquely activates
  async support, `lazy` and `Loading` are optional, transitions do not require a
  wrapper, and async work can enter the reactive graph from any computation.
- Therefore the async machinery cannot be cleanly removed by tree-shaking or
  shipped as an ecosystem-compatible addon. A separate `solidjs/sync-only`
  runtime would benchmark smaller but would not compose with much of the
  ecosystem.
- Ryan's rough size comparison: the same tiny signal counter that could come in
  just under 5 kB minzipped in Solid 1.x is just under 10 kB minzipped in Solid
  2.0 because almost 5 kB of async machinery remains in the baseline runtime.
- This is an intentional capability-for-minimum-size trade-off, not an accidental
  regression: the baseline runtime carries async behavior that otherwise appears
  only in substantially larger systems.

## Source statement (Ryan Carniato, July 16, 2026)

> There are different ways to keep libraries small. You can make them
> incrementally additive the way Preact ships a core and then install only the
> addons you want (hooks, signals, react-compat).
>
> You can have them treeshakable so you only pay for what you use. Solid 1.x was
> really good at that. Almost everything was gated, one install but only the code
> paths that could be hit based on imports are included. If you make certain
> global flags only writable there, code can be pruned at a conditional block
> level. This is super powerful when the boundary between features isn't so clean.
>
> The end result is both frameworks can make a counter example using signals in
> slightly under 5kb minzipped.
>
> Solid 2.0 unfortunately is a lot less treeshakeable because of how streamlined
> the API is. Because no single API determines whether there is async in the
> system.. no createResource/Async or `use`, lazy and Loading(Suspense) are
> completely optional, and no explicit transition wrapper, and no specific
> keyword like await, the async part isn't treeshakeable or addable. If a promise
> or async iterator makes it into the graph in anyway Solid handles it properly.
> And it can get in to the system anywhere that it makes sense to.
>
> So that same example now is just under 10kb minzipped because almost 5kb of
> async code can't be automatically dropped and it doesn't make sense to have
> special version for it because it wouldn't compose with the ecosystem, async
> like the passing of time itself is pretty unavoidable. So we won't be winning
> any of the toy app size comparisons anymore. But the runtime packs the atleast
> same capability of systems much much larger. So its a choice we will have to
> live with.
>
> Here was the journey:
> https://github.com/solidjs/solid/issues/2883

Ryan later added that this is also why he does not expect a `solidjs/sync-only`
variant: it might look good in benchmarks, but would be incompatible with much
of the ecosystem. In the same discussion, he agreed with the observation that
route splitting / multi-file publishing can lazy-load portions of a core runtime,
although not as granularly as a resumable runtime can chunk actions.

## Practical guidance

Do not recommend a hypothetical sync-only Solid 2.0 build as the normal answer
to bundle-size concerns. Instead:

1. Measure the production bundle rather than relying on toy-counter comparisons.
2. Treat the ~10 kB minimum discussed above as a deliberate baseline capability
   trade-off, not proof that the framework stopped caring about size.
3. Continue to tree-shake application and ecosystem code where normal ESM
   boundaries allow it.
4. Use route/code splitting where it helps real application loading behavior.
5. When comparing Solid 1.x and 2.0, distinguish **minimum runtime size** from
   **runtime capability included by default**.

## Related design thread

- Solid issue #2883: `https://github.com/solidjs/solid/issues/2883`
