# Persist

Durable state that survives the model's context window.

## Base durability mechanisms

- **Filesystem** — writing files the agent can re-read later
- **Git** — version-controlled state with history
- **Progress files** — scratchpad / TODO state across turns

The model **writes** to persistence and **reads** from it on subsequent turns. This is the practical mechanism for long-term memory in file-based harnesses.

## Why this is the cleanest long-term memory pattern

- **Plain text, plain files.** Auditable, diff-able, grep-able.
- **Portable across harnesses.** A markdown file is a markdown file.
- **No vendor lock-in.** Compare to opaque memory artifacts inside a closed harness.
- **Aligns with how engineers already work.** Git is already the durable state for code.

This is the answer to "how do I keep memory open?" → store it as files, version it with git, never put it behind an API.

## Patterns from Claude Code (relevant precedent)

- `CLAUDE.md` / `AGENTS.md` — auto-loaded context on session start.
- `.claude/skills/*/SKILL.md` — progressive-disclosure skill bodies, loaded on invocation.
- Wiki pattern (from user's global CLAUDE.md) — append-only log + topic pages.
- Auto-memory directory — typed memory files indexed by `MEMORY.md`.

## Audit-before-action durability

A pattern that prevents the most insidious failure mode of agentic systems: an action the agent *thinks* it took but didn't, or took silently with no record. The discipline:

1. **Persist the intent first.** Before calling the side-effecting tool, write `pending/<id>.json` with the full payload, timestamp, and reasoning.
2. **Execute.** Call the tool.
3. **Record the outcome.** Move `pending/<id>.json` → `done/<id>.json` (success) or `failed/<id>.json` (with error).

Benefits:
- **Crash recovery.** A killed runtime can re-scan `pending/` on restart and retry or escalate.
- **Audit trail.** Every action has a file with the full context — diff-able, grep-able, replay-able.
- **Deduplication.** Idempotency key in the filename prevents the same action firing twice.
- **Human review surface.** `pending/` is the queue a human reviews when in `draft` confidence mode (see `../control/principles.md`).

This is the persistence-layer answer to "how do I trust an agent with write actions?" The agent doesn't get trust; the file system does.
