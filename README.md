# Agent-Building Knowledge Base

Working notes for understanding how to build agentic workflows. The output of this exercise will be a **set of skills** for building/improving agentic systems, including a "starting skill" that bootstraps the design process.

> **How to build, not what to build.** Notes on the *shape* of agentic workflows — architecture patterns, skill design, memory ownership, failure modes. Not a library of pre-built agents.

## Why this exists

Goal: synthesize the best thinking from the public community on how agents should be built, then design the right shape of skills to make this knowledge actionable.

## Prerequisites

These notes are harness- and model-agnostic — use your preferred LLM with tool use and whatever harness works for you (Claude Code, openclaw/nanoclaw, hermes-agent, or your own).

Recommendation: build with the most capable model available (Opus/Sonnet class or equivalent), since design judgment benefits from headroom. Production can dispatch to smaller or public models per task class — cheaper tokens, narrower context windows, multi-modal where needed, and multi-LLM routing where each model handles what it's best at.

## Structure

Each folder corresponds to a dimension of agent theory. Files within capture principles, patterns, and tradeoffs — not anecdotes or implementation details.

```
workflow_skill/
├── README.md               ← you are here
├── SKILL.md                ← main skill: design-workflow (diagnostic + pattern catalog + routing)
├── architecture.md         ← synthesized 3-layer model + latent/deterministic principle
│
├── persona/
│   └── principles.md       multi-layer model (identity + soul + skin + steering), voice-DNA
├── memory/
│   └── principles.md       memory IS harness, types, compaction pipeline, search-augmented, providers
├── skill-design/
│   └── principles.md       method-call pattern, encoding spectrum, learning loop, diarization
├── tools/
│   └── principles.md            narrow fast tools beat god-tools
├── harness/
│   ├── principles.md       5-component model + harness shape (persistent vs container, multi-channel routing)
│   ├── context-injection/principles.md
│   ├── control/principles.md    compaction, orchestration, ralph loops, permission spectrum
│   ├── observe-verify/principles.md
│   └── persist/principles.md    files as the cleanest long-term memory
├── resolvers/
│   └── principles.md            routing tables for context, just-in-time injection
│
└── sub-skills/             ← invokable skills called directly or routed to by SKILL.md
    ├── scenarios/SKILL.md       enumerate every realistic input — happy path is not the spec
    ├── constraints/SKILL.md     define the mutable / immutable boundary
    ├── memory-system/SKILL.md   walk the seven memory questions
    ├── workflow-ux/SKILL.md     human-in-the-loop surface, latency, escalation
    ├── workflow-review/SKILL.md audit (5 dimensions) + debug (5 context-degradation modes)
    └── voice-dna/SKILL.md       clone a user's communication style from real samples
```

## Approach

An interpretation of what's most relevant right now for designing agentic workflows. Anthropic models (Claude Code in particular) are the concrete reference point for skill format and runtime details. The core theory — harness anatomy, memory ownership, execution patterns, design DNA — is model-agnostic and applies to any capable model with API access and tool use.

## Key conclusions so far

1. **The harness is the product.** The most consistent signal across theory and production systems.
2. **Memory and harness are inseparable.** Owning one means owning the other.
3. **Right architecture: lean runtime + content-rich markdown skills + deterministic tooling.** Variations exist in shape (e.g., container-per-execution) but the layering holds.
4. **Latent vs deterministic split is the most important boundary.** Wrong-side placement is the most common mistake.
5. **Persona is its own layer**, separate from operational identity, memory, and skills. Production systems treat it as stacked, independently editable data files.
6. **Skills come in (at least) three encodings** — method-call, declarative reference, branch+setup — each with different ergonomics.
7. **Compaction is a pipeline, not a single choice** — cheap deterministic trims run before costly model-generated compression. Claude Code's pipeline (Budget Reduction → Snip → Microcompact → Context Collapse → Auto-compact) is the canonical example; the question is which layers to include and in what order, not which single strategy to pick.
8. **Self-improving skills exist** but need scope boundaries to avoid drift and rule rot.
9. **Provider abstraction** is the practical mechanism behind model-agnostic claims; without it, model-agnostic is aspirational.
10. **"Agent workflow" is not one shape.** Six recurring patterns: pipeline, state machine, scheduled digest, REPL/ReAct, hydrated agent loop, full agent loop. Picking the wrong one is a top cause of fragile systems.
11. **Boil-the-ocean before architecture.** For natural-language workflows, exhaustive scenario enumeration is non-skippable. The happy path is never the spec.
12. **Constraints enable autonomy.** Every workflow needs an explicit mutable / immutable boundary. Ground truth and evaluation must live in the immutable zone, or the agent can "improve away" its own checks.
13. **Graduated trust, not "agent acts directly."** Confidence-routed branching (auto / draft / clarify) is a UX slice of a fuller permission spectrum (Plan / Default / AcceptEdits / Auto / DontAsk / BypassPermissions / Bubble). Deny-first evaluation and reversibility-weighted risk are the cross-cutting rules that keep the spectrum coherent.
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
25. **Foundational values precede architecture.** Five recur: human decision authority, safety/security/privacy, reliable execution, capability amplification, contextual adaptability. They trade off against each other; honest design names which it prioritizes and which it concedes rather than claiming to optimize all five.
26. **Extensibility is a four-mechanism surface** (MCP servers / plugins / skills / hooks), not one thing. Judgment → skill, integration → MCP, enforcement → hook, distributable bundle → plugin. Well-designed capabilities combine two or more.
