# Harness engineering

## What is a harness?

The harness is everything around the LLM that turns it into an agent. The model reasons and decides; the harness handles everything else.

An agent, by definition, is an LLM interacting with tools and other sources of data. There is always a system around the LLM to facilitate that interaction.

Even web search "built into" OpenAI/Anthropic APIs is a harness — a lightweight one — sitting in front of the model, calling search via tool calling.

**Scale matters:** production harnesses are substantial — often hundreds of thousands of lines of code — while the model is hosted separately. The harness is where the system's behavior actually lives.

## Harness components

The model sits in the middle. Five components surround it:

```
                    ┌─────────────────────┐
                    │  Context Injection  │
                    │ prompts, memory,    │
                    │ skills, conversation│
                    └──────────┬──────────┘
                               │
                               ▼
   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
   │   Control   │─────▶│    Model    │─────▶│    Tools    │
   │ compaction, │      │   reasons → │      │ bash, edit, │
   │ orchestra-  │      │   decides   │      │ grep, MCPs  │
   │ tion, ralph │      │             │◀─────│             │
   │   loops     │      │             │      │  results    │
   └─────────────┘      └──────┬──────┘      └─────────────┘
                               │  ▲
                       writes  │  │ reads        ┌──────────────────┐
                               ▼  │              │ Observe & Verify │
                    ┌─────────────────┐          │ screenshots,     │
                    │     Persist     │          │ tests, logs      │
                    │ filesystem,     │          └──────────────────┘
                    │ git, progress   │
                    │ files           │
                    └─────────────────┘
```

| Component | Role | Examples |
|-----------|------|----------|
| **Context Injection** | What the model sees on each turn | system prompts, memory recall, skill metadata, conversation history |
| **Control** | When/how the model is invoked | compaction triggers, multi-agent orchestration, ralph (self-referential) loops |
| **Tools** | How decisions become side effects | bash, built-in tool calls, MCP servers; results flow back into context |
| **Observe & Verify** | Closing the loop on tool outcomes | screenshots, test results, log scraping |
| **Persist** | Durable state outside the model's context | filesystem writes, git commits, progress / scratchpad files |

## Persistent agent vs container-per-execution

Beyond the components, the first shape decision is whether the runtime is one long-running process or an orchestrator that spawns containers per execution. The right choice follows the workload. Real-time conversational agents want persistent — no spawn overhead, low latency per turn. Multi-tenant or security-sensitive workflows (financial, healthcare, multi-channel coordination) want container-per-execution — strong isolation, crash-safe resumption, easier credential vaulting.

| | Persistent agent | Container-per-execution |
|--|------------------|-------------------------|
| **Shape** | Long-running process holds session state in memory | Stateless orchestrator spawns ephemeral container per task |
| **State recovery** | Restart loses in-process state unless explicitly persisted | Crash-safe: orchestrator can die and resume from checkpointed session ID |
| **Latency** | Low (no spawn overhead) | High (spawn + cold context per execution) |
| **Security** | Shared process, weaker isolation | Strong isolation per execution, easier credential vaulting |
| **Auditability** | Logs only | Each execution is a discrete unit with bounded scope |
| **Cost shape** | Predictable per-session | Pays per-execution overhead always |

## Multi-channel routing

Production agents often span multiple input/output channels — chat platforms, email, voice, CLI. This creates four recurring harness responsibilities: normalizing input across platforms that have different message shapes and attachment models; routing by channel context where Slack threads, Discord channels, and email threads need the right grouping key; formatting output per channel (markdown for Slack, plain text for SMS, rich blocks for Discord); and deciding whether the same user reaching the agent through different channels should share context.

This is a real harness responsibility that canonical framings underemphasize. The practical implication: workflow design should ask "what channels?" early — the answer changes output format, memory grouping, and the whole user-continuity model.

## Why harness anatomy is the foundation

Treating "the layers before generation" (the harness section) as *one of several* design considerations inverts the real relationship. **The harness is the primary thing**; workflow steps are one specific harness pattern (Control + Context Injection on a sequential plan).

Leading with harness anatomy gives a workflow skill a stable spine. Workflows, agents, ralph loops, deep agents are then **harness shapes** — not separate categories.
