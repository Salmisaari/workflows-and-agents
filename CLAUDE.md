# CLAUDE.md

Start here. Map of the repo and rules for where things go.

## Purpose

A skill set for designing agentic workflows. `SKILL.md` at the root is the entry point (`design-workflow`). Sub-skills in `sub-skills/` are invoked directly or routed to by the main skill. Layer folders hold the theory the skills stand on.

## Layout

| Path | Contains |
|------|----------|
| `README.md` | Human narrative: why this exists, conclusions |
| `SKILL.md` | Main skill — `design-workflow` entry point (diagnostic + pattern catalog + routing) |
| `architecture.md` | Cross-layer synthesis (3-layer model, pattern catalog, design DNA, stop-at-ambiguity) |
| `persona/`, `memory/`, `skill-design/`, `tools/`, `resolvers/`, `harness/` | Layer theory — `principles.md` per folder |
| `harness/context-injection/`, `harness/control/`, `harness/observe-verify/`, `harness/persist/` | 5-component anatomy — `principles.md` per folder |
| `sub-skills/<name>/SKILL.md` | Invokable sub-skills — `scenarios`, `constraints`, `memory-system`, `workflow-ux`, `workflow-review`, `voice-dna` |

## File conventions

- `principles.md` — synthesized theory for a named topic
- `SKILL.md` — invokable skill with frontmatter (name, description) and procedure body
- No author attributions in body (hurts downstream skill performance)

## Routing rules

- Cross-layer insight → `architecture.md`
- Layer-scoped insight → that layer's `principles.md`
- Component-scoped insight → that component's `principles.md`
- Invokable procedure → `sub-skills/<name>/SKILL.md`
- New top-level folder → update this file first, then create
- Narrative / conclusions → `README.md`, not here

## Read order

1. This file
2. `architecture.md`
3. Relevant folder based on the question
