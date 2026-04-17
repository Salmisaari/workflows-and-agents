# Persona

How an agent maintains a consistent identity, voice, and behavioral steering across turns and sessions. Distinct from memory of *the world*; this is memory *of the self*.

## Why persona is its own layer

Persona is not "a long system prompt." It's a structured concern with its own design choices, separate from:
- **Skills** (how to do things)
- **Memory** (what was learned about the world)
- **Context injection** (what the model sees this turn)
- **Operational identity** (what role/permissions the agent has)

Two failure modes appear when persona is implicit:
1. **Drift across sessions** — agent voice/tone changes turn by turn or day by day
2. **Persona overload in subagents** — parent's identity bleeds into task-bound children, contaminating their judgment

## The multi-layer persona model

Production systems (e.g., Hermes from Nous Research) treat persona as **stacked, independently editable data files**, assembled into the system prompt each turn:

```
┌─────────────────────────────────────────────────┐
│  Behavioral steering                            │  ← per-model overrides
│  ("with this provider, prefer absolute paths")  │
├─────────────────────────────────────────────────┤
│  Presentation / skin                            │  ← visual + textual brand
│  (name, banners, tone-of-UI, prompt symbols)    │
├─────────────────────────────────────────────────┤
│  Soul / character                               │  ← values, voice, dispositions
│  (SOUL.md — the user's persona contribution)    │
├─────────────────────────────────────────────────┤
│  Core identity                                  │  ← stable agent identity
│  ("you are Hermes, helpful, direct, ...")       │
└─────────────────────────────────────────────────┘
                       │
                       ▼
        Assembled into system prompt
        Cacheable (amortizes cost)
```

Each layer can be edited without touching the others. The whole stack is rebuilt every turn from files, so updates propagate immediately and can be swapped at runtime (`/skin ares` mid-conversation).

## Operational identity vs personality identity

Two distinct things are easily conflated:

| | Operational identity | Personality identity |
|--|----------------------|----------------------|
| **Encodes** | Role, permissions, capabilities, conventions | Voice, values, tone, dispositions |
| **Example file** | `CLAUDE.md` ("you can browse, schedule tasks; follow Slack format") | `SOUL.md` ("you are direct, you push back, you favor simplicity") |
| **Stable across users?** | Yes — defines the agent | No — should be customizable per user/deployment |
| **Required for every agent?** | Yes | Optional — many useful agents have only operational identity |

Some real-world systems (e.g., OpenClaw orchestrators) deliberately have **only operational identity** — there's no SOUL.md, no character file. The agent is a coordinator, not a personality. This is a valid choice when consistency-of-tone matters less than consistency-of-procedure.

## Voice-DNA: persona from observed behavior

A specialization of persona engineering: instead of authoring SOUL.md by hand, **derive it from a corpus of the user's own writing/speech**. The result is an agent that writes "in the user's voice."

Mechanism (general shape):
1. Sample a corpus of user-authored content (Slack messages, docs, emails).
2. Extract patterns: sentence length, hedge words, punctuation habits, opener/closer conventions, vocabulary range.
3. Encode the pattern as SOUL.md / voice-DNA file.
4. Inject as part of the persona stack.

This collapses the "teach the agent to sound like me" problem into the same persona-as-data architecture — no special runtime needed.

## Subagent persona policy

Two valid stances:

| Stance | Used by | Reasoning |
|--------|---------|-----------|
| **Subagents inherit no persona** | Hermes pattern | Children are task-bound; persona pollutes judgment, increases token cost, makes outputs less crisp |
| **Subagents inherit parent persona** | (Less common) | When the subagent's output is user-facing and tone consistency matters end-to-end |

Default toward **no inheritance** unless the subagent's output reaches the user directly. Internal-facing subagents (research, validation, summarization) work better with neutral, task-only prompts.

## Persistence mechanism

Persona must survive across sessions. The reliable pattern:

1. Persona files live on disk (`SOUL.md`, `skin.yaml`, etc.)
2. Harness reads files at session start AND on every turn (or rebuilds from cached version)
3. Whole stack composed into system prompt
4. System prompt is cached (provider-side cache) so the cost is amortized

This is the same architectural shape as long-term memory: **plain files, owned by the user, version-controlled, model-agnostic**. Persona is a special case of long-term memory whose subject is the agent itself.

## Open questions for the rewrite

- Does the workflow skill prescribe a persona layer at all, or only mention it as available?
- For use cases where the agent writes on behalf of a specific person: is voice-DNA replication worth a dedicated step in the starting skill?
- How does persona interact with multi-agent workflows where each agent has a different role? (Likely: each agent has its own persona stack; orchestrator has its own identity separate from any of them.)
- Should persona include negative space — explicit statements of what the agent will NOT do? (E.g., "you do not use emojis unless asked.")
