# Memory Principles

## Core thesis

Memory is not a plugin — it is what the harness *does*. Asking to plug memory into an agent harness is like asking to plug driving into a car. Managing context, and therefore memory, is a core capability and responsibility of the agent harness. Any design that treats memory as an external service it "calls into" is fighting the architecture.

## Memory types

| Type | Lives in | Updated by | Lifetime |
|------|----------|------------|----------|
| **Working memory** | Current turn's context | Each turn | One turn |
| **Short-term memory** | Conversation buffer | Harness, after each turn | One session |
| **Long-term memory** | External store (file, DB) | Harness, between sessions | Persistent |
| **Entity memory** | Per-user / per-thing record | Harness, when entity referenced | Persistent |
| **Time-sensitive memory** | Tagged with TTL | Harness, with expiry | Until stale |

The harness decides:
- What gets promoted from working → short-term → long-term.
- What survives compaction.
- What's queryable vs. recalled-by-default.
- How memory is presented (system prompt? tool result? message?).
- Whether the agent can edit its own memory / system instructions.

## Harness questions that *are* memory questions

Each of these is a harness implementation detail with memory consequences:

- How is `AGENTS.md` / `CLAUDE.md` loaded into context?
- How is skill metadata shown? (system prompt vs system message)
- Can the agent modify its own system instructions?
- What survives compaction, what's lost?
- Are interactions stored and queryable later?
- How is memory metadata presented to the agent?
- How is the working directory represented? How much filesystem is exposed?

If the skill talks about workflow design without addressing these, it's incomplete.

## Lock-in implications (recap from openness spectrum)

- **Stateless = portable.** Model swap is cheap when memory is yours.
- **Stateful API = sticky.** Conversation threads die at the provider boundary.
- **Closed harness = trapped.** Memory artifacts exist but are unreadable elsewhere.
- **Managed agent = owned by them.** No exit path.

## Compression strategies

When the conversation buffer approaches the context window, the harness must compact. The strategy is a meaningful harness choice:

| Strategy | Mechanism | Loses | Preserves |
|----------|-----------|-------|-----------|
| **Drop oldest** | Truncate past N turns | Everything beyond the window | Recency only |
| **Head/tail protect** | Keep system prompt + most recent N tokens; summarize the middle | Mid-conversation detail | Identity + recent state |
| **Iterative summary** | Old summary + new turns → new summary on each compression | Original phrasing | Cumulative intent and progress |
| **External recall** | Push old turns to searchable store; recall on demand via tool | In-context fluency on old material | Everything is recoverable if asked for |

Iterative summary preserves more information than head/tail across very long sessions but risks **summary drift** — each iteration is a lossy translation of the prior summary.

External recall (search-augmented memory, e.g., SQLite FTS5 over message history) is the most robust but requires the agent to know when to search — a judgment call that adds turns.

## Search-augmented memory

For sessions or domains where any prior interaction might become relevant later, persistent searchable storage is the answer:

- **Storage:** SQLite + FTS5, or a vector store, or both
- **Granularity:** per-turn, per-session, per-entity
- **Recall mechanism:** explicit tool the agent calls (e.g., `search_past_sessions("topic")`)
- **Indexing:** writes happen at turn-end; reads happen when the agent reaches for the tool

Search-augmented memory is the practical answer to "how do I have unbounded long-term memory without exhausting context?" Combined with file-based long-term memory (CLAUDE.md, MEMORY.md), it covers both the always-on and on-demand cases.

## Pluggable memory providers

Mature systems abstract memory behind a provider interface so the storage backend can be swapped (file → SQLite → external service like Honcho/Letta) without changing skill or runtime code. This is the same lesson as provider abstraction for models: avoid lock-in by isolating the integration in one place.

The constraint worth holding: regardless of backend, the agent should never directly write to a backend that can't be exported as plain files. If you can't `git diff` your memory, you don't really own it.
