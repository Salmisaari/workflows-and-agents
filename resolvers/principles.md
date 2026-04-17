# Resolvers

A resolver is a **routing table for context**: when task type X appears, load document Y first.

Skills tell the model *how*. Resolvers tell the model *what to load and when*.

## The built-in resolver: skill descriptions

Claude Code already ships with a resolver. Every skill has a `description` field, and the model matches user intent against those descriptions automatically.

> "You never have to remember that `/ship` exists. The description is the resolver."

The implication is that the description field is **load-bearing infrastructure**, not metadata. Sloppy descriptions break the resolver. Specific, intent-matched descriptions — with the right trigger words, named entities, named contexts — make it work.

## The bloated-CLAUDE.md anti-pattern

A real-world example: a `CLAUDE.md` grew to 20,000 lines — every quirk, every pattern, every lesson the author had encountered. Model attention degraded measurably. The fix was about 200 lines of pointers to documents. The resolver loads the right one when it matters. The 20,000 lines stayed accessible on demand without polluting the context window on every turn.

This is a concrete instance of **always-on injection vs. just-in-time injection** (see `../harness/context-injection/principles.md`):

- Always-on (system prompt, `CLAUDE.md`): token cost paid every turn → keep it small.
- Just-in-time (resolver-loaded docs, skill bodies): token cost paid only when relevant → can be large.

## The pattern, generalized

```
CLAUDE.md / AGENTS.md (always loaded, ~200 lines)
  ├─ pointer: "When changing prompts → read docs/EVALS.md"
  ├─ pointer: "When working on billing → read docs/BILLING.md"
  ├─ pointer: "When deploying → run /ship skill"
  └─ pointer: "When investigating an outage → run /investigate"
```

The pointers are the routing table. The model reads them every turn (cheap), but only loads the heavy docs when the trigger fires.

## Applying it

Keep always-on context lean and trigger-rich. A `SKILL.md` at ~200 lines is about right for the always-on slot — past that, it usually tries to do too much (preferences + design rules + review stages + interaction protocol bundled together).

To stay lean:

- Descriptions carry the triggers. The body stays short.
- Detailed material moves into linked documents loaded only when relevant (`harness.md`, `skills.md`, `architecture.md`).
- The resolver pattern applies *inside* the skill too: the skill body can dispatch to sub-procedures based on the kind of task it's handling.
