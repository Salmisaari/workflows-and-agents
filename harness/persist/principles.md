# Persist

Durable state that survives the model's context window. The agent writes to persistence; it reads from it on later turns. This is the practical mechanism for long-term memory in file-based harnesses.

## Base durability mechanisms

Three patterns cover almost everything:

- **Filesystem** — write files the agent can re-read later.
- **Git** — version-controlled state with full history.
- **Progress files** — scratchpad / TODO state tracked across turns.

## Why files are the cleanest long-term memory

Filesystem-backed memory is auditable, diff-able, grep-able. It's portable across harnesses — a markdown file is a markdown file. There's no vendor lock-in, because there's no vendor between the agent and the memory. And it aligns with how engineers already work: git is already the durable state for code.

This is the answer to "how do I keep memory open?" — store it as files, version it with git, never put it behind an API.

## The wiki pattern

One concrete shape worth naming: an append-only log paired with stable topic pages. Each event lands in the log on arrival; topic pages curate the durable state on topics that matter. The log is cheap to write and grep; the topic pages are what the agent reads on the next turn. Trade: raw recency for curated relevance.

## File-based IPC

When agents need to coordinate — parent ↔ child, agent ↔ orchestrator, agent ↔ scheduled task — real systems prefer file-based IPC over direct method calls. The agent writes `tasks/new-task.json` to a watched directory; the orchestrator polls or watches, validates deterministically, and executes. Shared state is never modified directly.

This produces a strongly-typed async boundary similar to microservice patterns, with three concrete benefits: every coordination action leaves an audit trail, any run can be replayed by re-reading the files, and the boundary is JSON — language-agnostic, not function calls.

## Audit-before-action durability

A pattern that prevents the most insidious failure mode of agentic systems: an action the agent *thinks* it took but didn't, or took silently with no record.

The discipline is three steps:

1. **Persist the intent first.** Before calling the side-effecting tool, write `pending/<id>.json` with the full payload, timestamp, and reasoning.
2. **Execute.** Call the tool.
3. **Record the outcome.** Move `pending/<id>.json` → `done/<id>.json` on success, or `failed/<id>.json` with the error.

What this buys:

- **Crash recovery.** A killed runtime can re-scan `pending/` on restart and retry or escalate.
- **Audit trail.** Every action has a file with full context — diff-able, grep-able, replay-able.
- **Deduplication.** Idempotency key in the filename prevents the same action firing twice.
- **Human review surface.** `pending/` is the queue a human reviews when in `draft` confidence mode (see `../control/principles.md`).

The principle: the agent doesn't get trust; the file system does.
