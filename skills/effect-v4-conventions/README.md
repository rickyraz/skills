# effect-v4-conventions

A reusable AI skill for writing and migrating TypeScript code that uses the Effect library (the `effect` npm package). Covers the full v3→v4 breaking-change surface so an assistant doesn't silently emit stale v3 APIs.

## Repository Layout

```text
skills/effect-v4-conventions/SKILL.md
skills/effect-v4-conventions/references/cause.md
skills/effect-v4-conventions/references/schema.md
skills/effect-v4-conventions/references/services.md
skills/effect-v4-conventions/references/import-api-map.md
skills/effect-v4-conventions/references/yieldable.md
skills/effect-v4-conventions/references/equality.md
skills/effect-v4-conventions/references/error-handling.md
skills/effect-v4-conventions/references/fiber-keep-alive.md
skills/effect-v4-conventions/references/fiberref.md
skills/effect-v4-conventions/references/forking.md
skills/effect-v4-conventions/references/generators.md
skills/effect-v4-conventions/references/layer-memoization.md
skills/effect-v4-conventions/references/runtime.md
skills/effect-v4-conventions/references/scope.md
skills/effect-v4-conventions/README.md
```

## Install Skill

Option 1: via `skills` CLI from repository URL

```bash
npx skills add https://github.com/rickyraz/skills --skill effect-v4-conventions
```

Option 2: install from direct skill path in repository URL

```bash
npx skills add https://github.com/rickyraz/skills/tree/main/skills/effect-v4-conventions
```

Option 3: clone repo, then copy only the skill folder into your tool's skill directory

```bash
git clone https://github.com/rickyraz/skills /tmp/<your-repo>
mkdir -p ~/.codex/skills
cp -R /tmp/<your-repo>/skills/effect-v4-conventions ~/.codex/skills/effect-v4-conventions
```

## Verify Installation

In a new AI assistant session, try:

```text
Use $effect-v4-conventions to migrate this file from Effect v3 to v4.
Flag every removed API and rewrite Context.Tag services to Context.Service.
```

If the skill is detected, the assistant will load:
- `skills/effect-v4-conventions/SKILL.md`
- the relevant file(s) under `skills/effect-v4-conventions/references/`

Common local skill directories by tool:
- Codex: `~/.codex/skills/`
- Claude Code: `~/.claude/skills/`
- Generic local setup: `~/.skills/`

## What This Skill Covers

- Master v3→v4 import path map (290 modules) and simple API renames
- `Context.Tag` / `Effect.Tag` / `Effect.Service` → unified `Context.Service`
- `Cause<E>` tree → flat `reasons` array; `*Exception` → `*Error`
- `Schema` migration: `transform`, `filter`, `optionalWith`, `pick`/`omit`, `extend`, variadic→array constructors
- `Ref`/`Deferred`/`Fiber` no longer `Effect` subtypes (`Yieldable` only)
- `catchAll*` → `catch*`, `catchSome*` → `catchFilter`/`catchCauseFilter`
- `FiberRef` removal → `Context.Reference`
- `Effect.fork` → `forkChild`, fork options object
- `Effect.gen(this, fn)` → `Effect.gen({ self: this }, fn)`
- Layer memoization sharing across `Effect.provide` calls
- `Runtime<R>` removal → `Effect.runForkWith`
- `Scope.extend` → `Scope.provide`
- Structural equality by default in `Equal.equals`

## Official References

- https://effect.website
- https://github.com/Effect-TS/effect

## Notes

- For local-only usage, you can skip CLI install and reference the absolute skill path directly.
- This skill assumes v4 unless the project's `package.json` clearly pins `effect@^3`.