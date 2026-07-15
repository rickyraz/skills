# orthogonal-blueprint

An agent skill for **orthogonal thinking** in product blueprints: stress-testing ideas and system architecture, escaping tunnel vision via a boring-first gate, six perspective axes, module orthogonality checks, reversibility maps, pre-mortems, and falsification criteria.

## Repository layout

```text
SKILL.md          # Primary instructions for the agent
reference.md      # Domain lenses and anti-patterns (reference)
```

There are no required scripts; the agent reads `SKILL.md` and follows links to `reference.md` when domain context is needed.

## Install skill

Using the `skills` CLI with the skill name:

```bash
npx skills@latest add https://github.com/rickyraz/skills --skill orthogonal-blueprint
```

Alternatively, copy this folder into your editor’s skills directory, for example `~/.cursor/skills/orthogonal-blueprint/` or `.cursor/skills/orthogonal-blueprint/` in a project repo.

## What this skill covers

- Phase 0: **boring solution gate** before shifting perspective
- Phase 1: surface problem vs underlying need
- Phase 2: **six axes** (inversion, abstraction, constraint, temporal, adversarial, cross-domain)
- Phase 3 & 3.5: module orthogonality and reversibility classification (Type 1 / Type 2)
- Phase 4 & 4.5: myopia audit, pre-mortem, and **falsification gate**
- Structured blueprint output template and sign-off checklist

## Additional reference

- Domain lenses (fintech, infra, hardware, web/mobile) and phrases to challenge: [`reference.md`](reference.md)

## Entry point

Start from [`SKILL.md`](SKILL.md) (frontmatter `name: orthogonal-blueprint` for discovery).
