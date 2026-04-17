# Resolvers

A resolver is a **routing table for context**: when task type X appears, load document Y first.

Skills tell the model *how*. Resolvers tell the model *what to load and when*.

## The built-in resolver: skill descriptions

Claude Code already has a resolver. Every skill has a `description` field. The model matches user intent to skill descriptions automatically.

> "You never have to remember that `/ship` exists. The description is the resolver."

Implication: the description field is **load-bearing infrastructure**, not metadata. Sloppy descriptions break the resolver. Specific, intent-matched descriptions (good triggers, named entities, named contexts) make the resolver work.

## The bloated-CLAUDE.md anti-pattern

A real-world example: a CLAUDE.md grew to 20,000 lines — every quirk, every pattern, every lesson the author had encountered. The model's attention degraded measurably. The fix was about 200 lines — just pointers to documents. The resolver loads the right one when it matters. The 20,000 lines of knowledge stayed accessible on demand, without polluting the context window on every turn.

This is a concrete instance of the **always-on injection vs just-in-time injection** distinction (see `../harness/context-injection/principles.md`):
- Always-on (system prompt, CLAUDE.md): pay token cost on every turn → keep it small.
- Just-in-time (resolver-loaded docs, skill bodies): pay token cost only when relevant → can be large.

## The resolver pattern, generalized

```
CLAUDE.md / AGENTS.md (always loaded, ~200 lines)
  ├─ pointer: "When changing prompts → read docs/EVALS.md"
  ├─ pointer: "When working on billing → read docs/BILLING.md"
  ├─ pointer: "When deploying → run /ship skill"
  └─ pointer: "When investigating an outage → run /investigate"
```

The pointers are the routing table. The model reads them on every turn (cheap), but only loads the heavy docs when the trigger fires.

## Why this matters for the workflow skill

The current `workflows` SKILL.md is itself ~200 lines and fairly content-dense. It probably *is* the right size for an always-on resolver — but it tries to do too much (preferences + design rules + review stages + interaction protocol). The rewrite could:

- Keep the SKILL.md short and trigger-rich (good descriptions, clear "when to use").
- Move detailed material into linked documents (e.g., `harness.md`, `skills.md`, `architecture.md`) loaded only when relevant.
- Use the resolver pattern *internally*: the skill body itself dispatches to sub-procedures based on the kind of workflow being designed.
