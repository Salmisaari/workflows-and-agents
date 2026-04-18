# Control

How and when the model gets invoked. The control layer is what turns a single model call into a workflow — it decides when to call, how to compact the prior context, whether to hand off to another agent, and whether the output flows to action or to review.

## Core patterns

Three control patterns show up everywhere:

- **Compaction** — collapse old context into a summary when the window fills.
- **Orchestration** — multi-agent or multi-step routing: which sub-agent runs next, when to merge.
- **Ralph loops** — self-referential iteration: the agent re-invokes itself with progress notes until a termination condition is met.

Five more surface in production systems:

- **Plan-then-execute** — separate planning model from execution model.
- **Critic / evaluator gate** — second model judges first model's output before commit.
- **Human-in-the-loop checkpoints** — pause for approval at risky boundaries.
- **Budget-bounded retry** — N attempts, then escalate.
- **Dynamic dispatch** — route to different sub-agents based on input type.

Each catches a different failure shape. Real systems stack them.

## Confidence-routed branching

A pattern that appears across real systems: the agent emits a structured judgment, and the deterministic layer routes on it — rather than the agent acting directly. Two modes catch two different failure classes.

### Action-routing (auto / draft / clarify)

Catches ambiguity at the *output*, right before a side-effecting action:

- **auto** — high confidence, execute without asking.
- **draft** — medium confidence, prepare the action and surface it for one-click approval.
- **clarify** — low confidence, ask a targeted question before proceeding.

The split lives in code, not in the prompt. The model returns `{confidence, action, reason}`; the runtime maps confidence band → branch. This keeps the latent/deterministic boundary clean: the model judges, the harness decides what to do with the judgment.

This is also where **action durability** is enforced (see `../persist/principles.md`): the `draft` branch persists the proposal *before* the human sees it, so an approved action can never silently disappear.

### Decision-point routing (proceed / stop)

Catches ambiguity at the *choice*, earlier in the loop — during planning or design, before any action has been taken. This addresses the empirically #1 expensive failure mode: the agent confidently going the wrong way at an ambiguous decision and burning ten minutes on the wrong thing (see `../../architecture.md` on stop-at-ambiguity).

```
At each planning / design decision:
   confidence clear → proceed
   confidence unclear → STOP.
                        Name the ambiguity.
                        List 2–3 plausible paths + implications.
                        Ask user.
```

The difference between the two modes:

| | Catches at | Prevents |
|--|-----------|----------|
| **Action-routing** | Output boundary | Wrong side-effect fires |
| **Decision-routing** | Planning / design step | Wrong path chosen silently, then built upon |

Most expensive agent failures can be prevented by one of these two. Skipping both is how you get confidently-wrong output delivered without friction.

## Permission modes and graduated trust

Confidence-routed branching addresses a single action. A fuller harness needs a *graduated trust spectrum* — a system-wide mode that shifts the default for every action, from "require approval for everything" to "act freely within boundaries." Claude Code's seven-mode taxonomy is the reference point worth learning:

| Mode | Default behavior |
|------|------------------|
| **Plan** | No execution; user must approve the plan before anything runs |
| **Default** | Standard interactive — most operations require approval |
| **AcceptEdits** | Auto-approves filesystem edits and safe shell commands; others prompt |
| **Auto** | ML-classifier evaluates anything not on the fast-path allow list |
| **DontAsk** | No prompting, but deny rules still enforced |
| **BypassPermissions** | Minimal prompting; only safety-critical checks remain |
| **Bubble** | Internal — subagent permission escalation surfaces to the parent terminal |

The auto/draft/clarify tiers above are a simplified UX slice of the same space: auto ≈ DontAsk (low friction, still bounded), draft ≈ Default (approval required), clarify ≈ Plan (nothing runs until discussion). The seven-mode version adds finer graduation (AcceptEdits for the common "edits are fine, network is not" case) and the bubble mechanism for subagent escalation.

Two design rules keep the spectrum coherent. **Deny-first evaluation** means that when an action matches both an allow rule and a deny rule, the deny wins — regardless of specificity. This is the usual access-control convention and it matters most under high-autonomy modes; a broadly-allowed Auto mode with a specific deny list is safer than the reverse, because the deny list is the bright line the agent cannot cross. **Reversibility-weighted risk** means not all actions deserve the same oversight — reversible actions (local file edits tracked in git, sandboxed runs) tolerate lighter gating than irreversible ones (external API writes, emails sent, payments executed). A mature permission system weights its prompting by reversibility, not by tool category; writing a file and wiring a payment shouldn't get the same prompt.

Together, the graduated spectrum + deny-first + reversibility weighting let a harness start conservative and earn trust over time. Longitudinal data on real Claude Code sessions shows users moving from ~20% auto-approval early on to 40%+ by hundreds of sessions — the permission system evolves with the relationship, it doesn't stay static.

## Concurrency and queueing

Whenever the harness is invoked by multiple users or event streams in parallel, scheduling becomes a control concern. Three patterns recur. **Queue with concurrency limit** bounds max parallel executions to control cost and rate limits. **Per-key serialization** ensures that within a single conversation or thread, turns don't race each other — the key is usually conversation-id or user-id. **Backpressure** defines what happens when the system is overloaded: drop, defer, or surface "I'll get back to you" to the user.

These belong squarely in the deterministic layer. Scheduling is not a judgment call the model should make — concurrency limits, queue ordering, and rate-limit decisions are rules the runtime enforces.

## Positioning sequential workflow

A sequential workflow — ordered steps with quality checks between them — is one **control pattern** among many, not the default. In workflow design:

- Surface ralph loops, parallel orchestration, and dispatch as siblings to sequential.
- Control choice is downstream of memory and context choices — you can't ralph-loop if context blows up after three iterations.
- Treat **confidence-routed branching** as the default for write actions where mistakes have real cost.
