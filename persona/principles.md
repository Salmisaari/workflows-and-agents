# Persona

How an agent maintains a consistent identity, voice, and behavioral steering across turns and sessions. Distinct from memory *of the world* — this is memory *of the self*.

## Why persona is its own layer

Persona is not "a long system prompt." It's a structured concern with its own design choices, separate from **skills** (how to do things), **memory** (what was learned about the world), **context injection** (what the model sees this turn), and **operational identity** (what role and permissions the agent has).

When persona is implicit, two failure modes appear:

1. **Drift across sessions** — the agent's voice and tone shift turn by turn.
2. **Persona overload in subagents** — the parent's identity bleeds into task-bound children, contaminating their judgment.

Treating persona as its own layer, with its own file(s) and its own update rules, prevents both.

## The multi-layer persona model

Production systems (e.g., Hermes from Nous Research) treat persona as **stacked, independently editable data files** assembled into the system prompt each turn:

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

## Operational identity vs. personality identity

Two distinct things get conflated. **Operational identity** encodes role, permissions, capabilities (`CLAUDE.md`: "you can browse, schedule tasks, follow Slack format"). **Personality identity** encodes voice, values, tone (`SOUL.md`: "you are direct, you push back, you favor simplicity").

| | Operational | Personality |
|--|-------------|-------------|
| **Encodes** | Role, permissions, capabilities | Voice, values, tone, dispositions |
| **Example file** | `CLAUDE.md` | `SOUL.md` |
| **Stable across users?** | Yes — defines the agent | No — customizable per user / deployment |
| **Required?** | Yes | Optional |

Some systems (OpenClaw orchestrators, for example) deliberately have **only operational identity** — no character file at all. The agent is a coordinator, not a personality. A valid choice when consistency-of-procedure matters more than consistency-of-tone.

## Voice-DNA: persona derived from observed behavior

A specialization: instead of authoring `SOUL.md` by hand, **derive it from a corpus of the user's own writing**. The result is an agent that writes in the user's voice, not an approximation of it.

### Four layers of voice

The naïve view says voice = vocabulary and sentence length. That produces caricature. A working voice profile covers four layers:

| Layer | What it encodes | Examples |
|-------|-----------------|----------|
| **Surface** | How they write | Sentence rhythm, lexical fingerprint, punctuation signatures |
| **Interpersonal** | How they relate | Directness, emotional texture, adaptation across audiences |
| **Cognitive** | How they think | Lead with conclusion or build to it, tolerance for ambiguity |
| **Depth** | Who they are | Attribution patterns, motivational signature, recurring values |

Only the surface layer alone produces caricature. All four together produce an agent that writes like the person on a good day.

### Constants vs. adaptations

Voice is a constant core plus channel-dependent deltas. A working profile maps both — the **constants** that appear everywhere (Slack, formal email, texts) and the **adaptations** that shift by audience and channel. The delta itself is part of the signature.

### Calibration: amplify / keep / soften

Not every real pattern should be amplified in generated output. The useful split is three-way:

- **Amplify** — patterns from the person's best communication (clear, warm, decisive). Weight higher.
- **Keep** — characteristic patterns that are neutral or positive (typos that signal urgency, bluntness that reads as confidence). Stripping them makes output sterile.
- **Soften** — patterns that appear under stress or fatigue (over-hedging, terseness-as-curtness). Reduce frequency; don't eliminate — eliminating breaks the voice.

The counter-intuitive move: many "imperfect" patterns belong in **Keep**, not Soften.

### Mechanism

1. Sample user-authored content across channels. Preserve samples exactly — typos are signal, not noise.
2. Analyze across the four layers.
3. Encode as a `SOUL.md` / voice-DNA file with constants, adaptations, signature patterns, calibration table.
4. Inject as part of the persona stack.
5. Update when the user edits generated drafts — edits are direct signal about where the profile is off.

A concrete skill implementation lives at `sub-skills/voice-dna/SKILL.md`.

## Subagent persona policy

When a parent agent spawns a subagent, does the persona inherit? Two valid stances:

- **No inheritance** (the Hermes pattern). Subagents are task-bound; persona pollutes judgment, costs tokens, makes outputs less crisp.
- **Full inheritance**. When the subagent's output is user-facing and tone consistency matters end-to-end.

Default to no inheritance unless the subagent's output reaches the user directly. Research, validation, and summarization subagents work better with neutral, task-only prompts.

## Persistence mechanism

Persona must survive across sessions. The reliable pattern:

1. Persona files live on disk (`SOUL.md`, `skin.yaml`).
2. Harness reads them at session start and rebuilds the stack every turn (or serves from cache).
3. Whole stack composes into the system prompt.
4. System prompt is cached provider-side so the cost is amortized.

Same architectural shape as long-term memory: **plain files, owned by the user, version-controlled, model-agnostic**. Persona is a special case of long-term memory whose subject is the agent itself.
