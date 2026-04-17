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

### What voice-DNA actually captures

The naïve view: voice = vocabulary and sentence length. A working voice-DNA model covers four layers:

| Layer | What it encodes | Examples |
|-------|-----------------|----------|
| **Surface** | How they write | Sentence rhythm, lexical fingerprint, typo and punctuation signatures, cadence |
| **Interpersonal** | How they relate | Pragmatic directness, emotional texture, audience adaptation between contexts |
| **Cognitive** | How they think | Lead with conclusion or build to it, tolerance for ambiguity, certainty calibration |
| **Depth** | Who they are | Attribution patterns, motivational signature, values that leak through repeatedly |

Only the surface layer produces a caricature. All four produce an agent that writes like the person on a good day.

### Constants vs. adaptations

Voice is a constant core plus channel-dependent deltas. A working profile maps both:

- **Constants** appear everywhere — Slack, formal email, texts. These are the signature.
- **Adaptations** are how register shifts by audience and channel. The delta itself is part of the signature.

### Calibration: amplify / keep / soften

Not every real pattern should be amplified in generated output. The useful split is three-way:

- **Amplify** — patterns from the person's best communication (clear, warm, decisive). Weight these higher.
- **Keep** — characteristic patterns that are neutral or positive (typos that signal urgency, bluntness that reads as confidence). Stripping them makes output sterile.
- **Soften** — patterns that appear under stress or fatigue the user would want less of (over-hedging, terseness-as-curtness). Reduce frequency, don't eliminate — eliminating breaks the voice.

The counter-intuitive move: many "imperfect" patterns belong in **Keep**, not Soften.

### Mechanism (general shape)

1. Sample user-authored content across channels (Slack, email, long-form). More channels = more dimensions resolved. Preserve samples exactly as written — typos are signal, not noise.
2. Analyze across the four layers above.
3. Encode the pattern as a SOUL.md / voice-DNA file with constants, adaptations, signature patterns, and a calibration table.
4. Inject as part of the persona stack.
5. Update the profile when the user edits generated drafts — edits are direct signal about where the profile is off.

This collapses the "teach the agent to sound like me" problem into the same persona-as-data architecture — no special runtime needed.

A concrete skill-file implementation is at `drafts/voice-dna.md`.

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
