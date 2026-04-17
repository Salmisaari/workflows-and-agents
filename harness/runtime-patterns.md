# Runtime Patterns

Beyond the diagram of *what's in* a harness (anatomy.md) and *how lean* it should be (runtime-sizing.md), real-world implementations reveal recurring patterns for *how the runtime is shaped*. These are the design choices visible in production agent systems.

## 1. Persistent agent vs container-per-execution

| | Persistent agent | Container-per-execution |
|--|------------------|-------------------------|
| **Shape** | Long-running process holds session state in memory | Stateless orchestrator spawns ephemeral container per task |
| **State recovery** | Restart loses in-process state unless explicitly persisted | Crash-safe: orchestrator can die and resume from checkpointed session ID |
| **Latency** | Low (no spawn overhead) | High (spawn + cold context per execution) |
| **Security** | Shared process, weaker isolation | Strong isolation per execution, easier credential vaulting |
| **Auditability** | Logs only | Each execution is a discrete unit with bounded scope |
| **Cost shape** | Predictable per-session | Pays per-execution overhead always |

The choice depends on what's optimized for. Real-time conversational agents → persistent. Multi-tenant or security-sensitive workflows (financial, healthcare, multi-channel coordination) → container-per-execution.

**Key insight:** *the runtime can stay lean even when each execution is heavy.* A 200-LOC orchestrator that spawns container agents stays lean at the runtime layer, even though each container runs a full agent loop inside.

## 2. Multi-platform / multi-channel routing

Production agents often operate across multiple input/output channels (chat platforms, email, voice, CLI). The harness must:

- **Normalize input** — different platforms have different message shapes, formatting expectations, attachment models
- **Route by channel context** — Slack threads ≠ Discord channels ≠ Email threads; the agent needs the right grouping key
- **Format output per channel** — markdown for Slack, plain text for SMS, rich blocks for Discord
- **Preserve continuity across channels** — if the same user reaches the agent through different channels, do they share context?

This is a real harness responsibility that canonical framings underemphasize. Implication: the workflow-design skill should ask "what channels?" early in the design process — it changes everything from output format to memory grouping.

## 3. Concurrency and queueing

The harness is often invoked concurrently (multiple users, multiple incoming events). Design choices:

- **Queue with concurrency limit** — bound max parallel executions to control cost and rate limits
- **Per-key serialization** — within a single conversation/thread, serialize so turns don't race
- **Backpressure** — when overloaded, drop, defer, or surface "I'll get back to you"

These belong in the deterministic layer (latent vs deterministic principle: scheduling is not a judgment call).

## 4. IPC / async coordination

When agents need to coordinate (parent ↔ child, agent ↔ orchestrator, agent ↔ scheduled task), real systems lean on **file-based IPC** rather than direct method calls:

- Agent writes `tasks/new-task.json` to a watched directory
- Orchestrator polls / watches, deterministically validates, executes
- Agent doesn't directly modify shared state

This produces a strongly typed async boundary, similar to microservice patterns. Benefits: auditable (every coordination action leaves a file), debuggable (replay by re-reading files), language-agnostic (the boundary is JSON not function calls).

## 5. Compression as a runtime responsibility

When context approaches the model's window, the harness must compact. Real strategies:

| Strategy | How it works | Best for |
|----------|--------------|----------|
| **Drop oldest** | Discard turns past N | Short-horizon tasks, when old turns truly don't matter |
| **Head/tail protection** | Always keep system prompt + most recent N tokens; compress the middle | Most general workflows |
| **Iterative summary** | Old summary + new turns → new summary, cycle | Long sessions where prior context informs current decisions |
| **External recall** | Push old turns to searchable store; recall on demand via tool | Open-ended sessions where any prior turn might matter |

The compression strategy is a harness choice with significant downstream effects on agent behavior. It's also where memory ownership matters most (see `../memory/principles.md`): if compression happens server-side behind a closed API, you can't read what was kept or dropped, and you can't migrate.

## 6. Provider abstraction

A serious harness allows model-swapping without code changes. The pattern:

- Internal message representation is the canonical form
- Per-provider adapters translate to/from each provider's API (Anthropic Messages, OpenAI Chat Completions, etc.)
- Tool schemas normalized across providers
- Per-provider prompt-engineering quirks isolated in adapter layer

This is the practical mechanism behind "be model agnostic" — without provider abstraction, model-agnostic is aspirational, not real.

## 7. Multi-model dispatch by task

A serious harness rarely uses one model for everything. Different tasks have different cost/latency/quality trade-offs, and a runtime that can dispatch to the right model per call captures large efficiency gains:

| Task class | Typical model choice | Why |
|------------|---------------------|-----|
| **Entity extraction, classification, routing** | Smallest available (Haiku, Gemini Flash, GPT-mini) | Cheap, fast, accuracy is high enough on narrow tasks |
| **Heavy synthesis, judgment, reasoning** | Largest available (Opus, GPT-5, Gemini Pro) | The latent-space work where capability matters |
| **Bulk batch processing** (e.g., re-classifying thousands of items) | Mid-tier with batch API | Cost dominates; latency irrelevant |
| **Vision / document parsing** | Vision-capable provider | Capability gating, not preference |
| **Long-context recall** | Gemini long-context, or model with cheapest input cost | Context window dictates choice |

Implementation looks like a `dispatch(task_type, payload) → model_id` function in the runtime. Skill metadata can declare preferred model class; the runtime resolves to a concrete model.

Combined with provider abstraction (section 6), this gives the harness genuine model independence — and the ability to upgrade individual task classes as new models drop without rewriting skills.

The anti-pattern: hard-coding the model name in skill files. That's the shape that ages worst.
