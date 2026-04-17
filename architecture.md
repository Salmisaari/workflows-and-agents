# Architecture (Synthesized)

The synthesis of two foundational framings: "own your harness / own your memory" and "lean runtime, content-rich skills." This is the working architectural model for the rewritten skill.

## The three layers

```
┌──────────────────────────────────────────────────────────────┐
│  CONTENT-RICH SKILLS                                         │
│  Markdown procedures: judgment, process, domain knowledge    │
│  Method-call shape (TARGET, QUESTION, DATASET → behavior)    │
│  ~90% of the value lives here                                │
│  Inherently portable (just files, version-controlled)        │
└──────────────────────────────────────────────────────────────┘
                              ▲
                              │ invoked by
                              │
┌──────────────────────────────────────────────────────────────┐
│  LEAN RUNTIME (HARNESS)                                      │
│  ~200 LOC. JSON in, text out.                                │
│  Does four things only:                                      │
│    1. Run the model in a loop                                │
│    2. Read/write your files                                  │
│    3. Manage context (compaction, injection)                 │
│    4. Enforce safety                                         │
│  Read-only by default. Open. Owns the memory.                │
└──────────────────────────────────────────────────────────────┘
                              ▲
                              │ calls
                              │
┌──────────────────────────────────────────────────────────────┐
│  DETERMINISTIC APPLICATION                                   │
│  Purpose-built tooling: QueryDB, ReadDoc, Search, Timeline   │
│  Narrow, fast, idempotent. Same input → same output.         │
│  Latency target: ~100ms per call, not 2-15s MCP round-trips. │
└──────────────────────────────────────────────────────────────┘
```

### The lean runtime in practice

The sub 200-LOC figure isn't aspirational. A runtime that reads/writes files, manages context, and runs a model loop genuinely fits in that budget — the rest is markdown skills outside the runtime. The lean property also survives heavy-execution shapes: a 200-LOC orchestrator that spawns container-per-execution agents stays lean at the runtime layer even though each container runs a full agent loop inside. What stays small is the runtime itself, not the total system.

The anti-pattern is the opposite: 40+ tool definitions eating half the context window, "god-tools" via MCP with multi-second round-trips per call, REST API wrappers that turn every endpoint into a separate tool. The concrete contrast: Chrome MCP doing screenshot → find → click → wait → read takes 15 seconds; a Playwright CLI doing each step in 100ms is 75× faster. The lesson generalizes — software doesn't have to be precious anymore. Build exactly the narrow, fast tool you need; don't reach for the kitchen-sink integration when a 50-line CLI does it.

## How this satisfies both framings

| Concern | "Own your memory" view | "Lean runtime" view | This architecture |
|---------|-----------------------|----------------------|-------------------|
| Where does long-term memory live? | In the harness, owned by you | In skill files | Skill files = your memory; harness owns the directory |
| Portable across models? | Required | Implicit (markdown is portable) | ✅ |
| Lock-in risk? | Avoid closed harnesses / managed agents | Don't use MCP god-tools either | ✅ |
| Where does intelligence live? | (Not addressed directly) | Latent space (the model) | Skills frame *how* the model thinks |
| Where does precision live? | (Not addressed directly) | Deterministic code | Application layer |

The two framings aren't in conflict — a lean runtime with content-rich skills is what an "open harness" looks like in practice.

## Model independence

"Model-agnostic" is easy to claim and hard to make real. Two practical mechanisms carry the weight.

**Provider abstraction** is what lets you swap the model without rewriting code. An internal canonical message format sits at the center, with per-provider adapters translating to/from each API (Anthropic Messages, OpenAI Chat Completions, Gemini). Tool schemas normalize across providers; prompt-engineering quirks stay isolated in the adapter layer. Without this, "model-agnostic" is aspirational — you can point at the swap, but you can't do it.

**Multi-model dispatch by task** picks up from there. A serious harness rarely uses one model for everything — entity extraction, heavy reasoning, bulk batch, vision, and long-context recall all have different cost/latency/quality profiles.

| Task class | Typical choice | Why |
|------------|----------------|-----|
| Entity extraction, classification, routing | Smallest available (Haiku, Flash, mini) | Narrow-task accuracy is enough |
| Heavy synthesis, judgment, reasoning | Largest available | Latent-space work where capability pays off |
| Bulk batch processing | Mid-tier with batch API | Cost dominates, latency irrelevant |
| Vision / document parsing | Vision-capable provider | Capability gating, not preference |
| Long-context recall | Long-context model, cheapest input | Context window dictates choice |

Implementation is a `dispatch(task_type, payload) → model_id` function. Skill metadata can declare a preferred class; the runtime resolves to a concrete model. Combined with provider abstraction, this gives the harness real model independence — new models drop, individual task classes upgrade, skills stay unchanged.

The anti-pattern: hard-coding model names in skill files. That's the shape that ages worst.

## The latent vs deterministic principle (cross-cutting)

Every step in your system is one of two things. Confusing them is the most common architectural mistake.

| | Latent | Deterministic |
|--|--------|----------------|
| **What** | Model reads, interprets, decides | Same input → same output |
| **Strength** | Judgment, synthesis, pattern recognition | Trust, scale, repeatability |
| **Examples** | Reading a profile and noticing the gap between what someone *says* and what they *build* | SQL queries, sorting, arithmetic, combinatorial optimization |
| **Failure mode** | Hallucination at scale (e.g., "seat 800 people at a banquet") | Brittleness when the world changes |

Push intelligence **up** into skills (latent). Push execution **down** into deterministic tooling. The harness is the interface between them.

When you do this:
- Every model upgrade automatically improves every skill (judgment gets better).
- The deterministic layer stays perfectly reliable (no regression risk from model changes).
- Skills become **permanent upgrades**.

## Workflow design is three-dimensional

A naive workflow treatment is one-dimensional (steps with handoffs). This architecture says workflow is actually three-dimensional:

1. **Where does this step live?** (latent / deterministic / harness)
2. **Is this work codifiable as a reusable skill?** (one-off vs codify-and-cron)
3. **What context does this step need at this moment?** (resolver question)

Any process-level design skill should make these three questions explicit.

## Pattern catalog

"Agent workflow" is not one shape. From studying real systems, the design space collapses to roughly six recurring patterns. The starting skill should diagnose which one fits, not assume "ReAct loop with tools" by default.

| Pattern | Shape | When it fits |
|---------|-------|--------------|
| **Pipeline** | Fixed sequence of steps; output of N → input of N+1; no branching | Repeatable transformations with stable structure (extract → classify → enrich → store) |
| **State machine** | Explicit states + transitions; agent advances state on each turn; resumable | Multi-turn workflows with clear stages (e.g., receipt: uploaded → categorized → assigned → approved) |
| **Scheduled digest** | Cron fires; agent reads sources, writes report, optionally pings human | Periodic synthesis with no real-time interaction (daily metrics, weekly review) |
| **REPL / ReAct** | User asks → model thinks → calls tool → reads result → answers; one-shot or short loop | Ad-hoc questions where the answer is in a queryable source |
| **Hydrated agent loop** | Pre-hydrate entity context, then agent loop over a tool set scoped to that entity | Entity-centric work (one supplier, one customer) where context is bounded |
| **Full agent loop** | Open-ended turn loop with broad tool access, persistent session, compaction strategy | Genuinely open-ended assistance (Claude Code, customer ops triage) |

### Diagnostic flow

A starting skill can route between these with four questions:

```
Q1. Is the work triggered by a user message, or by a schedule/event?
    → user message      → continue to Q2
    → schedule/event    → Pipeline or Scheduled digest

Q2. Does it complete in one turn, or does it span multiple?
    → one turn          → REPL / ReAct
    → multiple          → continue to Q3

Q3. Is there a fixed set of states the work moves through?
    → yes               → State machine
    → no                → continue to Q4

Q4. Is the work scoped to a single entity (supplier, customer, project)?
    → yes               → Hydrated agent loop
    → no                → Full agent loop
```

Picking the wrong pattern is a leading cause of agent projects feeling fragile. A scheduled digest dressed up as a full agent loop will be expensive and brittle. A genuinely open-ended assistant constrained to a state machine will frustrate users.

## Boil-the-ocean design heuristic

Natural-language workflows attract scope creep because users describe the *happy path* and assume edge cases will surface during implementation. They won't — until production, expensively.

The heuristic: **before designing, exhaustively enumerate the scenario space.** If a user says "build an agent for handling returns," the design conversation has not started yet. The design conversation is:

```
Returns scenario tree:
├── Return accepted
│   ├── Refund (full / partial)
│   ├── Store credit / coupon
│   ├── Exchange (same SKU, different size)
│   └── Exchange (different SKU, price delta handling)
├── Return rejected
│   ├── Outside return window
│   ├── Item damaged by customer
│   ├── Final-sale item
│   └── Missing components
├── Return in limbo
│   ├── Customer never ships item
│   ├── Item lost in transit
│   └── Partial return (some items returned, some kept)
└── Return triggers other work
    ├── Inventory restock decision
    ├── Quality flag if pattern emerges
    └── Refund payment-method routing
```

Now the design conversation has substance. Now you can pick a pattern (state machine, almost certainly), define states explicitly, and decide which branches are in-scope vs deferred.

The discipline:
1. **Generate the scenario tree before architecture.** A LLM is excellent at this — prompt it: "list every way this workflow can branch, including weird ones."
2. **Mark each branch in-scope / out-of-scope / deferred.** Don't silently drop branches.
3. **For deferred branches, define the escape hatch.** What does the agent do when it hits one? Hand off, refuse, log-and-continue.
4. **Re-run scenario generation after the v1 ships.** Real usage surfaces branches the design missed.

The starting skill should make this enumeration step *non-skippable* for any workflow more complex than a pipeline.

## Multi-agent coordination (the other axis)

Pattern catalog above describes the *shape of one workflow*. A different axis is how *multiple agents coordinate* when a single workflow isn't the right granularity. Three recognized shapes:

| Pattern | How agents relate | When it fits |
|---------|-------------------|--------------|
| **Supervisor / orchestrator** | Central agent decomposes tasks, routes to workers, holds global state | Clear task decomposition; human oversight needed; most common starting point |
| **Peer-to-peer / swarm** | Agents hand off directly to each other; no central controller | Exploratory work where rigid planning is counterproductive |
| **Hierarchical** | Layered: strategy → planning → execution; each layer talks only to adjacent ones | Genuinely complex projects with structural depth |

Selection is about **coordination needs**, not organizational metaphor. Critical constraint: each sub-agent should run in a **clean context window focused on its subtask**, not carry accumulated context from siblings.

Anti-pattern to avoid: over-decomposition. "A 10-step pipeline with 10 agents spends more tokens on handoffs than on actual work." Start with one agent; add a second only when context isolation genuinely helps.

This axis is orthogonal to the 6 execution shapes above. A state-machine workflow can be run by a single agent or by a supervisor-worker pair; the coordination choice is downstream.

## Token-budget reality

One empirical finding worth internalizing across every skill: **"more tokens ≈ better performance" explains roughly 80% of the variance in agent quality.** The model getting more careful, more relevant context is the dominant lever. Model choice, prompt phrasing, tool selection — all matter, but less.

Implications:
- **Latency vs. quality is a real trade-off, not a pretend one.** A hydrated-loop agent that pre-fetches structured context will outperform a ReAct agent that discovers the same context via tool calls — because it gives the model more tokens up front.
- **Cost/quality trade-off is explicit.** Bigger context = higher per-call cost. Workflows that run hot (high frequency) can't afford the heavy-context approach; workflows that run cold (strategic synthesis) should lean into it.
- **The UX layer pays.** Any compression / trimming / "let the agent fetch" choice is trading tokens for latency/cost. Name the trade each time, don't optimize it invisibly.
- **Evaluate at production-realistic context sizes.** A workflow that works in a clean 2k-token eval can collapse at 40k tokens in production. Test under the conditions that will actually run.

This is why workflow UX is a design axis of its own: responsiveness and accuracy pull in opposite directions on the token axis, and the right answer is use-case-specific.

## Constraints principle (mutable vs immutable boundary)

The core rule: **the agent's autonomy must be defined by what it cannot change.**

Worked example: a research agent has full freedom inside `train.py` — model architecture, hyperparameters, training loop, anything. But it cannot modify:
- `prepare.py` (the data pipeline)
- The evaluation harness
- The dependency graph

Why this matters: **constraints enable autonomy.** If the agent could rewrite the evaluation harness, "did the run improve" becomes a meaningless question — the agent could always claim improvement. If it could rewrite `prepare.py`, runs become incomparable.

Generalized: every agent system needs an explicit boundary between mutable surface and immutable scaffolding.

| Mutable (agent's playground) | Immutable (the floor it stands on) |
|------------------------------|------------------------------------|
| The artifact under construction (code, plan, draft) | Evaluation criteria & test data |
| Working notes, scratchpad | Tool definitions, schemas |
| Choice of approach within scope | Out-of-scope boundary itself |
| Skill invocation order | Skill bodies (in production runs) |

The principle to enforce in workflow design:

1. **Name the immutable surfaces explicitly.** "The agent can edit X. The agent cannot edit Y. The runtime enforces this." If you can't draw the line, you don't have a workflow yet.
2. **Ground truth lives in the immutable zone.** Tests, eval rubrics, success criteria. Anything the agent could "improve away" is not a real check.
3. **Make the constraint visible to the agent.** The system prompt should include the constraints. The agent operating against an invisible fence is a failure mode.
4. **Revisit constraints, but as a human-driven step.** Don't let the agent loosen its own boundaries; that's how scope creep becomes silent.

This pairs with the "audit-before-action" pattern in `harness/persist/principles.md` (the file system is the immutable record of what happened) and with confidence-routed branching in `harness/control/principles.md` (the routing logic is immutable; the model only contributes a confidence score).

## Design DNA

Four anti-over-engineering principles that should govern *both* the skill files you write (meta) and the workflows the skills describe (content).

1. **Think Before Acting** — surface assumptions explicitly. If uncertain, ask. Multiple interpretations → present, don't pick silently. Unclear → stop.

2. **Simplicity First** — minimum capability that serves the scenarios you committed to. No speculative features, no abstractions with one caller, no error handling for conditions that can't occur. *"Would a senior engineer say this is overcomplicated?"* If yes, delete.

3. **Surgical Changes** — when iterating a workflow (v1 → v2), touch only what the change requires. Don't refactor adjacent things. Don't "improve" what wasn't asked for. Every changed line should trace to the specific delta.

4. **Goal-Driven Execution** — every workflow needs a verifiable success criterion *before* you build it. "Make it work" is not a criterion. Without a test, you can't iterate — you're just moving noise around.

These pair productively with the other principles:

- **Boil-the-ocean + Simplicity First.** Enumerate exhaustively to understand the scope; build minimally for what ships. Enumeration is free; implementation is expensive. The tension is intentional — you can only build minimally if you've first seen maximally.
- **Constraints + Goal-Driven Execution.** The verifiable success criterion lives in the immutable zone. The agent can improve the workflow, but not the test.
- **Stop-at-ambiguity + Think Before Acting.** The runtime mechanism that enforces the first principle (see below).

## Stop-at-ambiguity principle

The #1 empirically expensive agent failure mode: the agent confidently picks the wrong path at an ambiguous decision point. Ten minutes of work, start over. The fix is a first-class runtime discipline, not a prompt suggestion.

```
At every decision point during planning or design:
  Agent asks itself: "Can I answer this definitively from context?"

  yes → proceed
  no  → STOP. Name the ambiguity explicitly.
        List the 2–3 plausible paths and their implications.
        Surface to the user. Do not proceed until answered.
```

Distinct from confidence-routed branching on write actions (see `harness/control/principles.md`) — this catches the error *before* any work happens, at the planning/design layer where wrong paths are invisible until they've cost you.

Required ingredients:
- **Confidence calibration.** Agent explicitly rates its confidence at each decision point.
- **Plausible-alternatives listing.** Naming 2–3 paths forces the model to actually think vs. picking one. Often the act of listing reveals which is right.
- **Visibility in system prompt.** The agent must know "stop at ambiguity" is expected behavior, not an apology.

Diagnostic question for any post-mortem: *"Was there a decision point the agent should have stopped at but didn't?"* Sibling failure class to the five context-degradation modes (see `harness/observe-verify/principles.md`).
