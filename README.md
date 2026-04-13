# react-view-transitions

A reusable AI skill for implementing, debugging, and explaining React View Transitions, including React release-channel constraints (Stable vs Canary/Experimental).

## Install Skill

Option 1: via `skills` CLI

```bash
npx skills add https://github.com/rickyraz/react-view-transitions
```

Option 2: clone manually into your tool's local skills directory

```bash
mkdir -p ~/.skills
git clone git@github.com:rickyraz/react-view-transitions.git ~/.skills/react-view-transitions
```

If you are using Codex specifically, common equivalents are:

```bash
npx skills add https://github.com/rickyraz/react-view-transitions --agent codex
mkdir -p ~/.codex/skills
git clone git@github.com:rickyraz/react-view-transitions.git ~/.codex/skills/react-view-transitions
```

## Verify Installation

In a new AI assistant session, try:

```text
Use $react-view-transitions to debug why my list reorder animation becomes a cross-fade.
```

If the skill is detected, the assistant will use `SKILL.md` and the internal reference in `references/react-view-transitions-architecture.md`.

## React Canary Notes (April 2026)

- `<ViewTransition />` and `addTransitionType` are currently available in Canary/Experimental channels, not in stable React 19.x.
- React docs for `<ViewTransition />` are still marked as `Canary`.
- The latest stable docs currently show React `19.2`.

To try the React View Transition API:

```bash
npm install react@canary react-dom@canary
```

Make sure `react` and `react-dom` are on the same channel (do not mix stable with canary/experimental).

If your project stays on stable React, use the browser API `document.startViewTransition()` directly.

## Check React Versions in a Project

```bash
npm ls react react-dom
```

## Official References

- https://react.dev/reference/react/ViewTransition
- https://react.dev/blog/2025/04/23/react-labs-view-transitions-activity-and-more
- https://react.dev/versions
- https://react.dev/community/versioning-policy
