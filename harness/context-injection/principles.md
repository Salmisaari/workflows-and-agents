# Context Injection

What the harness puts in front of the model on each turn. Every injection is a deliberate budget choice — stale context is worse than no context, and the prefix you keep stable is the prefix that gets cached.

## Five layers, one breakpoint

What the model sees on each turn breaks into five layers, ordered from most stable to most volatile:

- **System** — the agent's identity and behavior baseline. The runtime's hosted system prompt (Claude Code's, or whatever your harness ships with), plus a project-level `CLAUDE.md`, plus persona docs that give a named agent its voice. Changes maybe once a week.
- **Tools** — capabilities available this session. Built-in tools, MCP servers, anything loaded at session start. Changes per session.
- **Conversation** — accumulated turns in this session. The state that has grown since the session started. Changes every turn, but only by appending.
- **Cache affinity** — not content; a decision. Where does the prefix that is stable across turns end, and the prefix that is volatile per turn begin? The breakpoint sits between Tools and Conversation by default: everything above it pays once and is read at a fraction of cost for the rest of the session; everything below it pays full input cost every turn.
- **Messages** — the new input the user just typed. Always volatile by definition; this is what the model is actually responding to.

The four content layers are the obvious decomposition; the breakpoint is the design decision. Move it up (so tools or system content becomes volatile per turn) and you've thrown away the cache. Move it down (so part of conversation becomes part of the stable prefix) and you're pinning information the model doesn't need to read on most turns.

## Stable prefix is the cheapest performance lever you have

Prompt caches reward stability. Anthropic's prompt cache — and equivalents on other providers — reads a cached prefix at a fraction of full input cost and a fraction of full input latency. Break-even is typically two to three reads; after that every turn that hits the cache is nearly free.

That makes the system and tools layers a strategic asset, not just configuration. A typed system-prompt structure with named parts (identity, guidelines, environment, tools, custom instructions, last-turn instructions) keeps edits localized and the prefix bit-identical across turns when nothing relevant changed. A free-form string concatenation does the opposite — small accidental variations (a re-formatted timestamp, a swapped paragraph order, a list reordered for "clarity") invalidate the cache and you pay full freight on every subsequent turn.

The discipline parallels what `../../tools/principles.md` advocates for tool surfaces, applied to the system prompt: treat it as a stable contract, not a place to scribble per-turn context.

## Where does memory live?

The five-layer framing forces a design decision the simpler list lets you dodge: recalled long-term memory has two homes, and you have to pick.

In the **system layer** (stable), memory is baked into `CLAUDE.md` or a persona doc and read for free after the first turn — but the same context shows up whether the user asks about it or not. Between **conversation and messages** (volatile), memory is retrieved per turn based on the new input — vector search, recent-summary recall, entity-keyed lookup. Targeted, but pays full input cost every turn and crosses the cache breakpoint.

The right answer depends on hit rate. Memory the agent uses on most turns belongs in the system layer; memory the agent uses on a fraction of turns belongs in the volatile layer where it can be retrieved selectively. Putting low-hit-rate memory in the system layer wastes tokens on every turn; putting high-hit-rate memory in the volatile layer pays the cache penalty needlessly.

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

Pre-hydration is a specific case of the volatile-memory pattern above — the runtime decides per turn what entity context to inject between conversation and messages, fully aware that the injection crosses the cache breakpoint. The cost is the cache miss; the benefit is structured density and reliability.

Why this beats letting the model fetch:

- **Latency.** One parallel fetch beats N sequential tool calls.
- **Token efficiency.** Pre-hydrated structured data is denser than tool-result sprawl.
- **Reliability.** The model can't forget to look something up.
- **Composability.** Hydration logic is deterministic and testable.

The cost: the runtime has to know *which entity to hydrate* before the main model runs. A cheap extraction step (regex, classifier, or small-model call) identifies the entity, then hydrates, then calls the main model.

This is one concrete shape of the **resolver** idea (see `../../resolvers/principles.md`): the harness decides what context the model needs and front-loads it.
