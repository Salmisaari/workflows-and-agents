# Workflows for Agents

A skill set and theory base for designing agentic workflows. The main skill (`SKILL.md`) walks a design sequence — enumerate scenarios, set constraints, pick a memory architecture, design the interaction surface, review before ship. The theory folders are the load-bearing ideas those skills stand on.

> **How to build, not what to build.** This repo is about the *shape* of agentic workflows — architecture patterns, skill design, memory ownership, failure modes. Not a library of pre-built agents.

## Why this exists

Working notes evolving over time. The goal is fewer unexplainable agent failures by setting the right foundations early — explicit scenario enumeration, a clean mutable / immutable boundary, memory that persists as plain files you can diff, graduated trust instead of "agent acts directly." Most agent problems aren't model problems; they're architecture decisions that were never made deliberately.

The same foundation scales from customer-facing work to internal operations. Workflows built on it have handled:

- Automating accounting
- Order handling
- Resolving customer-support situations
- Taking internal notes on tasks
- Coordinating priorities across teams
- Reading analytics and flagging anomalies
- Shipping new features

The shape of each workflow changes; the design sequence doesn't.

## How to use this repo

1. Read [`CLAUDE.md`](CLAUDE.md) for the repo map, then [`architecture.md`](architecture.md) for the synthesis.
2. Invoke [`SKILL.md`](SKILL.md) (`design-workflow`) to design a new workflow end-to-end — it routes to the sub-skills below.
3. Open any theory folder directly when a specific question is already narrowed.

Harness- and model-agnostic — any capable model with tool use will do.

## Sub-skills

Invokable procedures, each with numbered steps, decision rules, and an output contract.

| Skill | Use when |
|-------|----------|
| [`scenarios`](sub-skills/scenarios/SKILL.md) | Enumerating the full range of realistic inputs. The happy path is never the spec. |
| [`constraints`](sub-skills/constraints/SKILL.md) | Defining the mutable / immutable boundary. Ground truth and evaluation must live in the immutable zone. |
| [`memory-system`](sub-skills/memory-system/SKILL.md) | Designing the memory architecture — what persists, what compacts, what drops, where it lives. |
| [`workflow-ux`](sub-skills/workflow-ux/SKILL.md) | Designing the human-in-the-loop surface — latency disclosure, wait states, confidence routing, escalation. |
| [`workflow-review`](sub-skills/workflow-review/SKILL.md) | Auditing (5 dimensions) or debugging (5 context-degradation modes + ambiguity). |
| [`voice-dna`](sub-skills/voice-dna/SKILL.md) | Cloning a user's communication style from real samples across four voice layers. |

## Theory layers

One `principles.md` per folder. Read the folder when the question is in that layer.

| Layer | Covers |
|-------|--------|
| [`persona/`](persona/principles.md) | Identity stack (core / soul / skin / steering), voice-DNA, subagent persona policy. |
| [`memory/`](memory/principles.md) | Memory types, compaction pipeline (5 layers), search-augmented recall, provider abstraction. |
| [`skill-design/`](skill-design/principles.md) | Method-call skills, encoding spectrum, learning loop, diarization, extensibility space. |
| [`tools/`](tools/principles.md) | Narrow fast tools beat god-tools. Tool descriptions are prompts, not docs. |
| [`harness/`](harness/principles.md) | The 5-component anatomy, persistent vs container-per-execution, sandboxes, multi-channel routing. |
| [`resolvers/`](resolvers/principles.md) | Routing tables for context, just-in-time injection, description-as-infrastructure. |

## Harness anatomy (5 components)

The model sits in the middle; five components surround it. Each maps to a common failure mode.

| Component | Role |
|-----------|------|
| [`context-injection/`](harness/context-injection/principles.md) | What the model sees on each turn — prompts, memory, skill metadata, conversation. Pre-hydration pattern. |
| [`control/`](harness/control/principles.md) | When and how the model is invoked — compaction, orchestration, ralph loops, confidence-routed branching, 7-mode permission spectrum. |
| [`observe-verify/`](harness/observe-verify/principles.md) | Closing the loop on tool outcomes — 5 validation dimensions + 5 context-degradation modes. |
| [`persist/`](harness/persist/principles.md) | Durable state — filesystem, git, audit-before-action, `pending/` queues, file-based IPC. |

*(Tools are the fifth component; their principles live at [`tools/`](tools/principles.md) as a peer layer.)*

## Core ideas

**Architecture.**
- Agent = model + harness. **The harness is the product** — ~90% of the value lives in content-rich skills above a lean (~200 LOC) runtime.
- **Latent vs. deterministic is the most important split.** Push judgment up into skills; push execution down into deterministic tooling.
- **Memory is the harness**, not a feature you bolt on. Plain files you can `git diff` beat a vendor API.
- **Token quality is ~80% of agent quality.** Every latency/cost trade-off should name what it gives up on the token axis.

**Design discipline.**
- **Boil-the-ocean scenarios before architecture.** The happy path is never the spec.
- **Constraints enable autonomy.** The agent can't improve away its own tests if ground truth lives in the immutable zone.
- **Stop-at-ambiguity is a runtime discipline**, not a prompt suggestion. The #1 expensive failure is confidently going the wrong way at a fork.

**Execution.**
- **Six recurring workflow shapes** — pipeline, state machine, scheduled digest, REPL/ReAct, hydrated agent loop, full agent loop. Picking the wrong one is the top cause of fragile systems.
- **Compaction is a pipeline, not a strategy.** Cheap deterministic layers run before costly model-generated ones.
- **Graduated trust beats "agent acts directly."** Confidence-routed branching (auto / draft / clarify) sits inside a 7-mode permission spectrum with deny-first evaluation and reversibility-weighted risk.
- **Audit-before-action.** Persist intent before side effects — crash-safety, dedup, and human-review surface fall out for free.

**Failure modes.**
- **Context degradation** explains most "the agent went wrong" incidents: lost-in-middle, poisoning, distraction, confusion, clash. Matching fixes: write / select / compress / isolate.
- **Maintainability is the silently-decaying dimension.** Skill-file sprawl, opaque memory choices, untracked prompt tweaks kill the workflow before a user complaint surfaces.

## Repo structure

```
workflow_skill/
├── CLAUDE.md               repo map + where-things-go rules
├── SKILL.md                main skill: design-workflow (diagnostic + routing)
├── architecture.md         cross-layer synthesis
│
├── persona/                identity stack, voice-DNA
├── memory/                 types, compaction pipeline, providers
├── skill-design/           method-call, encoding spectrum, learning loop
├── tools/                  narrow fast tools beat god-tools
├── resolvers/              routing tables for context
├── harness/
│   ├── principles.md       5-component anatomy + sandboxes + multi-channel
│   ├── context-injection/  what the model sees
│   ├── control/            when/how the model is invoked
│   ├── observe-verify/     closing the loop
│   └── persist/            durable state
│
└── sub-skills/             invokable skills
    ├── scenarios/
    ├── constraints/
    ├── memory-system/
    ├── workflow-ux/
    ├── workflow-review/
    └── voice-dna/
```

## File conventions

- `principles.md` — synthesized theory for a named topic.
- `SKILL.md` — invokable skill with frontmatter (`name`, `description`) and a procedure body.
- No author attributions inside skill or principle bodies (they hurt downstream skill performance).

## Prerequisites

Any capable model with tool use. Claude Code is the concrete reference for skill format and runtime details, but the core theory — harness anatomy, memory ownership, execution patterns, design DNA — is model-agnostic.
