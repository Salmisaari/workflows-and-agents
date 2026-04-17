# CLAUDE.md

Start here. Map of the repo and rules for where things go.

## Purpose

Theory for designing agentic workflows. Will be distilled into 6 skill files (`design-workflow`, `memory-system`, `scenarios`, `constraints`, `workflow-review`, `workflow-ux`) that replace `~/.claude/skills/workflows/SKILL.md`.

## Layout

| Path | Contains |
|------|----------|
| `README.md` | Human narrative: why this exists, conclusions |
| `architecture.md` | Cross-layer synthesis (3-layer model, pattern catalog, design DNA, stop-at-ambiguity) |
| `persona/`, `memory/`, `skills/`, `tools/`, `resolvers/` | Layer theory — `principles.md` per folder |
| `harness/*.md` | Runtime-shape dimensions — one file per dimension (anatomy, openness-spectrum, runtime-sizing, runtime-patterns) |
| `harness/context-injection/`, `harness/control/`, `harness/observe-verify/`, `harness/persist/` | 5-component anatomy — `principles.md` per folder |

## File conventions

- `principles.md` — synthesized theory for a named topic
- No author attributions in body (hurts downstream skill performance)

## Routing rules

- Cross-layer insight → `architecture.md`
- Layer-scoped insight → that layer's `principles.md`
- Component-scoped insight → that component's `principles.md`
- New top-level folder → update this file first, then create
- Narrative / conclusions → `README.md`, not here

## Read order

1. This file
2. `architecture.md`
3. Relevant folder based on the question
