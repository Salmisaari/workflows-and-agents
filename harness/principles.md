# Harness engineering

## What is a harness?

**Agent = Model + Harness.** If you're not the model, you're the harness.

The harness is everything around the LLM that turns it into an agent. The model reasons and decides; the harness handles everything else — state across turns, tool execution, enforceable constraints, access to anything outside its context window. A raw model takes text/images/audio and returns text; that's it. Everything else is harness — including the lightweight shim behind "built-in" web search in an API, which is still a tool-calling loop with a different label.

**Scale matters:** production harnesses are substantial — often hundreds of thousands of lines of code — while the model is hosted separately. The harness is where the system's behavior actually lives.

## Working backwards from desired behavior

Each component of a harness earns its place by filling a gap between what the model does natively and what an agent needs to do. The design move is: name the behavior, then ask what piece of the harness has to provide it. Durable state across sessions calls for filesystem + git. Autonomous code execution calls for bash + a sandbox. Access to post-training knowledge calls for web search or MCPs. Holding performance on long tasks calls for compaction and progressive-disclosure skills. Long-horizon coherence calls for planning, verification, and — when needed — ralph loops.

The five-component anatomy below is the same set reorganized around *where in the loop* each piece lives (what the model sees, when it runs, what it calls, how it verifies, what persists). Working backwards from behavior is the move when designing a new harness; the five-component cut is the move when debugging an existing one.

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

## Sandboxes and default environments

When the agent runs code or shell commands, it does so in an environment the harness configures — and those configuration choices shape behavior as much as the model or tool set does. The harness is responsible for isolation (running code where a mistake can't damage the host, via containers, ephemeral VMs, command allow-listing, and network egress controls); default tooling (what's pre-installed on turn one is what the agent reaches for — language runtimes, git, test runners, browsers, package managers); and observability handles (stdout/stderr, exit codes, diffs, screenshots, logs — without these the agent can't close the loop on its own actions).

A subtle point about default tooling: a model that could theoretically shell out to install Python won't, if the task is short-horizon and Python isn't already there. Defaults are prompts. The observability side matters just as much — the environment must surface what happened after each tool call, or the agent's next decision is made on stale information.

Sandboxing lives naturally inside the container-per-execution shape but isn't limited to it; persistent agents also run in configured environments. The rule of thumb: for any tool that could touch shared state or the network, ask what prevents a confused agent from causing irreversible damage, and bake the answer into the environment rather than relying on prompting.

## Multi-channel routing

Production agents often span multiple input/output channels — chat platforms, email, voice, CLI. This creates four recurring harness responsibilities: normalizing input across platforms that have different message shapes and attachment models; routing by channel context where Slack threads, Discord channels, and email threads need the right grouping key; formatting output per channel (markdown for Slack, plain text for SMS, rich blocks for Discord); and deciding whether the same user reaching the agent through different channels should share context.

This is a real harness responsibility that canonical framings underemphasize. The practical implication: workflow design should ask "what channels?" early — the answer changes output format, memory grouping, and the whole user-continuity model.

## Why harness anatomy is the foundation

Treating "the layers before generation" (the harness section) as *one of several* design considerations inverts the real relationship. **The harness is the primary thing**; workflow steps are one specific harness pattern (Control + Context Injection on a sequential plan).

Leading with harness anatomy gives a workflow skill a stable spine. Workflows, agents, ralph loops, deep agents are then **harness shapes** — not separate categories.
