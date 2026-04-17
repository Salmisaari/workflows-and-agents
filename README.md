# Agent-Building Knowledge Base

Working notes for understanding how to build agentic workflows. The output of this exercise will be a **set of skills** for building/improving agentic systems, including a "starting skill" that bootstraps the design process.

> **How to build, not what to build.** Notes on the *shape* of agentic workflows — architecture patterns, skill design, memory ownership, failure modes. Not a library of pre-built agents.

## Why this exists

Goal: collect theory of agent construction from current best sources (articles + real-world implementations), then design the right shape of skills to make this knowledge actionable.

The current `workflows` skill at `~/.claude/skills/workflows/SKILL.md` is the starting point. It conflates several concerns and uses overly rigid interaction gating. The rewrite (or split into multiple skills) will be informed by what's captured here.

## Prerequisites

These notes are harness- and model-agnostic — use your preferred LLM with tool use and whatever harness works for you (Claude Code, openclaw/nanoclaw, hermes-agent, or your own).

Recommendation: build with the most capable model available (Opus/Sonnet class or equivalent), since design judgment benefits from headroom. Production can dispatch to smaller or public models per task class — cheaper tokens, narrower context windows, multi-modal where needed, and multi-LLM routing where each model handles what it's best at.

## Structure

Each folder corresponds to a dimension of agent theory. Files within capture principles, patterns, and tradeoffs — not anecdotes or implementation details.

```
workflow_skill/
├── README.md               ← you are here
├── architecture.md         ← synthesized 3-layer model + latent/deterministic principle
├── persona/
│   └── principles.md       multi-layer model (identity + soul + skin + steering), voice-DNA
├── memory/
│   └── principles.md       memory IS harness, types, compression, search-augmented, providers
├── skills/
│   └── principles.md       method-call pattern, encoding spectrum, learning loop, diarization
├── tools/
│   └── notes.md            narrow fast tools beat god-tools
├── harness/
│   ├── anatomy.md          5-component model: context-injection, control, tools, observe, persist
│   ├── openness-spectrum.md  control surface from open → managed
│   ├── runtime-sizing.md   lean runtime + content-rich skills
│   ├── runtime-patterns.md   persistent vs container, multi-channel, IPC, compression, providers
│   ├── context-injection/notes.md
│   ├── control/notes.md    compaction, orchestration, ralph loops
│   ├── observe-verify/notes.md
│   └── persist/notes.md    files as the cleanest long-term memory
└── resolvers/
    └── notes.md            routing tables for context, just-in-time injection
```

## Approach

An interpretation of what's most relevant right now for designing agentic workflows. Anthropic models (Claude Code in particular) are the concrete reference point for skill format and runtime details. The core theory — harness anatomy, memory ownership, execution patterns, design DNA — is model-agnostic and applies to any capable model with API access and tool use.

## Key conclusions so far

1. **The harness is the product.** Both articles and both implementations confirm this.
2. **Memory and harness are inseparable.** Owning one means owning the other.
3. **Right architecture: lean runtime + content-rich markdown skills + deterministic tooling.** Variations exist in shape (e.g., container-per-execution) but the layering holds.
4. **Latent vs deterministic split is the most important boundary.** Wrong-side placement is the most common mistake.
5. **Persona is its own layer**, separate from operational identity, memory, and skills. Production systems treat it as stacked, independently editable data files.
6. **Skills come in (at least) three encodings** — method-call, declarative reference, branch+setup — each with different ergonomics.
7. **Compression strategy is a meaningful harness choice** — drop / head-tail / iterative summary / external recall, each with different tradeoffs.
8. **Self-improving skills exist** but need scope boundaries to avoid drift and rule rot.
9. **Provider abstraction** is the practical mechanism behind model-agnostic claims; without it, model-agnostic is aspirational.
10. **"Agent workflow" is not one shape.** Six recurring patterns: pipeline, state machine, scheduled digest, REPL/ReAct, hydrated agent loop, full agent loop. Picking the wrong one is a top cause of fragile systems.
11. **Boil-the-ocean before architecture.** For natural-language workflows, exhaustive scenario enumeration is non-skippable. The happy path is never the spec.
12. **Constraints enable autonomy.** Every workflow needs an explicit mutable / immutable boundary. Ground truth and evaluation must live in the immutable zone, or the agent can "improve away" its own checks.
13. **Confidence-routed branching (auto / draft / clarify)** is a saner default than "agent acts directly" for any write action with real cost.
14. **Audit-before-action** persistence: write the intent to a `pending/` file before the side-effecting tool call. Crash-safety, dedup, and human-review surface all fall out for free.
15. **Pre-hydration beats letting the model fetch.** When the entity is known, hydrate once and inject; don't make the model spend turns on tool calls for context it should have had on turn one.
16. **Multi-model dispatch by task class** is how serious harnesses get cost/latency right. Hard-coded model names in skill files age worst.
17. **Context degradation is the hidden failure mode.** Five named patterns (lost-in-middle, poisoning, distraction, confusion, clash) explain most "the agent went wrong" incidents. Each has a matching fix (write / select / compress / isolate).
18. **Tool descriptions are prompts.** Not docs. They directly shape agent behavior. Verb-noun naming, single clear purpose, recovery-focused error messages.
19. **Coordination is a second axis.** Execution shape (pipeline / state-machine / ...) is orthogonal to agent coordination (supervisor / peer-to-peer / hierarchical). Start with one agent; multi-agent only when context isolation earns its keep.
20. **Token quality is ~80% of agent quality.** "More careful context = better accuracy" is the dominant empirical lever. Every latency/cost trade-off should name what it's giving up on the token axis.
21. **The design DNA** (think before acting, simplicity first, surgical changes, goal-driven execution) applies at two levels: the skill files themselves, and the workflows those skills describe. Both levels rot without it.
22. **Stop-at-ambiguity is a first-class runtime discipline**, not a prompt suggestion. The #1 expensive failure mode is agents confidently going the wrong way at an ambiguous decision point. Required: confidence calibration, plausible-alternatives listing, stop-behavior explicit in system prompt.
23. **Review has two modes, one skill.** Proactive audit (5 dimensions: strategic fit / UX / technical / maintainability / safety) and reactive debug (5 context-degradation modes + ambiguity). Both answer "which dimension failed?" — different entry points to the same diagnostic.
24. **Maintainability is the silently-decaying dimension** for agent workflows. Skill-file sprawl, opaque memory choices, untracked prompt tweaks kill the workflow before a user complaint ever shows up.

## Open questions before designing the skill set

1. **Audience scope:** is the skill set for people **building harnesses**, **writing skill files for an existing harness like Claude Code**, or both?
2. **Skill granularity:** one umbrella `workflows` skill, or several focused skills (`harness-design`, `memory-design`, `persona-design`, `skill-design`, `architecture-review`)?
3. **Starting skill:** what does the entry point look like? A diagnostic that asks the user about their goal and routes to the relevant focused skills?
4. **Interaction style:** the existing skill uses forced BIG/SMALL gating and mandatory pauses. What's the right replacement — guided-but-skippable? Pure reference? Conversational?
5. **Persona depth:** does the skill set need a dedicated persona/voice-DNA skill, or is it a section within a broader skill?
6. **Self-modifying mechanism:** do we want any of the resulting skills to learn from feedback, or keep them all static for v1?

## Status

- [x] Theory captured from articles 1, 2
- [x] Real-world patterns captured from openclaw, hermes-agent
- [x] Productive tensions surfaced and synthesized
- [x] Pattern catalog distilled from real-world agents
- [x] Boil-the-ocean heuristic added (scenario enumeration before architecture)
- [x] Constraints principle captured (mutable / immutable boundary)
- [x] Confidence-routed branching, audit-before-action, pre-hydration, multi-model dispatch folded in
- [x] 5 context-degradation failure modes folded into observe-verify (lost-in-middle, poisoning, distraction, confusion, clash)
- [x] Tool-design principles folded into tools (consolidate, verb-noun, recovery errors)
- [x] Multi-agent coordination axis (supervisor / peer-to-peer / hierarchical) added to architecture
- [x] Token-budget reality principle captured (~80% of variance from context quality)
- [x] Skill set agreed: 6 topic-named skills (design-workflow, memory-system, scenarios, constraints, workflow-review, workflow-ux)
- [x] Design DNA (four principles) + Stop-at-ambiguity principle captured in architecture.md
- [x] Decision-point branching captured in harness/control/notes.md (companion to action-routing)
- [x] Five proactive validation dimensions captured in harness/observe-verify/notes.md
- [x] Theory pass complete — ready for skill drafting
- [ ] Draft the 6 skill files
- [ ] Replace / refactor existing `workflows` SKILL.md
