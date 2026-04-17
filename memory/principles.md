# Memory

Memory is not a plugin — it is what the harness *does*. Asking to plug memory into an agent harness is like asking to plug driving into a car. Managing context, and therefore memory, is a core capability and responsibility of the harness itself.

Any design that treats memory as an external service the agent "calls into" is fighting the architecture.

## Memory types

Five shapes cover almost all memory use. Each has its own lifetime, update rhythm, and place to live:

| Type | Lives in | Updated by | Lifetime |
|------|----------|------------|----------|
| **Working memory** | Current turn's context | Each turn | One turn |
| **Short-term memory** | Conversation buffer | Harness, after each turn | One session |
| **Long-term memory** | External store (file, DB) | Harness, between sessions | Persistent |
| **Entity memory** | Per-user / per-thing record | Harness, when entity referenced | Persistent |
| **Time-sensitive memory** | Tagged with TTL | Harness, with expiry | Until stale |

The harness decides the flow between them: what gets promoted from working → short-term → long-term, what survives compaction, what's queryable vs. always-present, how memory is presented (system prompt, tool result, or message), and whether the agent can edit its own memory at all.

## Harness questions are memory questions

Questions that look like implementation details are actually memory design decisions in disguise:

- How is `AGENTS.md` / `CLAUDE.md` loaded into context?
- How is skill metadata shown — system prompt or system message?
- Can the agent modify its own system instructions?
- What survives compaction, and what gets dropped?
- Are interactions stored and queryable later?
- How is the working directory represented? How much filesystem is exposed?

A workflow design that addresses execution but not these is incomplete.

## Lock-in implications

Memory design determines portability:

- **Stateless harness, memory-on-disk** — portable. Model swap is cheap.
- **Stateful provider API** (OpenAI Threads, Anthropic sessions) — sticky. Conversation threads die at the provider boundary.
- **Closed harness** — memory artifacts exist, but can't be read elsewhere. Trapped.
- **Managed agent platform** — memory owned by the vendor. No exit path.

The principle: if you can't `git diff` your memory, you don't really own it.

## Compression strategies

When the conversation buffer approaches the context window, the harness must compact. The choice is meaningful — it shapes what the agent can reason about later:

| Strategy | Mechanism | Loses | Preserves |
|----------|-----------|-------|-----------|
| **Drop oldest** | Truncate past N turns | Everything beyond the window | Recency only |
| **Head/tail protect** | Keep system prompt + recent N tokens; summarize the middle | Mid-conversation detail | Identity + recent state |
| **Iterative summary** | Old summary + new turns → new summary on each compression | Original phrasing | Cumulative intent and progress |
| **External recall** | Push old turns to searchable store; recall on demand via tool | In-context fluency on old material | Everything, if the agent asks |

Iterative summary preserves more than head/tail across very long sessions but risks **summary drift** — each pass is a lossy translation of the prior one. External recall is the most robust but requires the agent to know when to search — a judgment call that costs turns.

## Search-augmented memory

For sessions where any prior interaction might become relevant later, persistent searchable storage is the answer. Storage is typically SQLite + FTS5, a vector store, or both. Granularity is per-turn, per-session, or per-entity depending on recall patterns. The agent pulls by invoking a tool (`search_past_sessions("topic")`); writes happen at turn-end.

Combined with file-based long-term memory (`CLAUDE.md`, `MEMORY.md`), this covers both the always-on and on-demand cases.

## Pluggable memory providers

Mature systems abstract memory behind a provider interface so the backend can be swapped (file → SQLite → Honcho / Letta) without changing skill or runtime code. Same lesson as provider abstraction for models: isolate the integration in one place, avoid lock-in.

Regardless of backend, the agent should never directly write to a store that can't be exported as plain files.
