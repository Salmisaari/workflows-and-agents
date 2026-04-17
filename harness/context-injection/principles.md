# Context Injection

What the harness puts in front of the model on each turn. Every injection is a deliberate budget choice — stale context is worse than no context.

Four sources combine into what the model sees:

- **Prompts** — system prompt, task framing.
- **Memory** — recalled long-term entries, summaries of prior turns.
- **Skills** — skill metadata (names and descriptions of available skills).
- **Conversation** — message history, or its compacted summary.

## Workflow-level vs. harness-level framing

A workflow-level rule: each step gets exactly the context it needs. Not the full history — a summary of what came before, plus the raw input it must act on.

The harness-level version of the same rule: **every injection costs tokens, and tokens degrade attention**. Give the model what it needs; no more.

## Pre-hydration

A pattern from production agents. When the user references an entity (a supplier, a customer, a SKU), don't make the model fetch it via tool calls. Hydrate it once at the start of the turn, inject it as a structured profile, let the model read:

```
User: "what's going on with Acme this month?"
        │
        ▼
[runtime extracts entity: supplier="acme"]
        │
        ▼
[runtime fetches profile, KPIs, recent threads — in parallel]
        │
        ▼
[injects ENTITY_PROFILE block into system message]
        │
        ▼
Model sees full hydrated context on turn 1, no tool calls needed
```

Why this beats letting the model fetch:

- **Latency.** One parallel fetch beats N sequential tool calls.
- **Token efficiency.** Pre-hydrated structured data is denser than tool-result sprawl.
- **Reliability.** The model can't forget to look something up.
- **Composability.** Hydration logic is deterministic and testable.

The cost: the runtime has to know *which entity to hydrate* before the main model runs. A cheap extraction step (regex, classifier, or small-model call) identifies the entity, then hydrates, then calls the main model.

This is one concrete shape of the **resolver** idea (see `../../resolvers/principles.md`): the harness decides what context the model needs and front-loads it.
