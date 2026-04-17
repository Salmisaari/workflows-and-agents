# Control

How and when the model gets invoked. Three core patterns exist; more variations surface in practice.

## Core control patterns

- **Compaction** — collapse old context into a summary when the window fills.
- **Orchestration** — multi-agent or multi-step routing (which sub-agent runs next, when to merge).
- **Ralph loops** — self-referential iteration: agent re-invokes itself with progress notes until a condition is met.

## Other control patterns worth capturing

- **Plan-then-execute** — separate planning model from execution model.
- **Critic / evaluator gate** — second model judges first model's output before commit.
- **Human-in-the-loop checkpoints** — pause for approval at risky boundaries.
- **Budget-bounded retry** — N attempts then escalate.
- **Dynamic dispatch** — route to different sub-agents based on input type.

## Confidence-routed branching

A pattern visible in real systems where the agent emits a structured judgment that the deterministic layer routes on, rather than the agent acting directly. Two companion modes catch different failure classes.

### Action-routing (auto / draft / clarify)

Catches ambiguity at the *output*, right before a side-effecting action:

- **auto** — high confidence, execute without asking
- **draft** — medium confidence, prepare action and surface for one-click approval
- **clarify** — low confidence, ask a targeted question before proceeding

The split lives in code, not in the prompt. The model returns `{confidence, action, reason}`; the runtime maps confidence band → branch. Keeps the latent/deterministic boundary clean: the model judges, the harness decides what to do with the judgment.

This is also where **action durability** is enforced (see `../persist/principles.md` on audit-before-action) — the `draft` branch persists the proposal *before* the human sees it, so an approved action can never silently disappear.

### Decision-point routing (proceed / stop)

Catches ambiguity at the *choice*, earlier in the loop — during planning or design, before any action has been taken. This is the discipline that addresses the empirically #1 expensive failure mode: the agent confidently going the wrong way at an ambiguous decision and burning ten minutes on the wrong thing (see `../../architecture.md` Stop-at-ambiguity principle).

```
At each planning / design decision:
   confidence clear → proceed
   confidence unclear → STOP.
                        Name the ambiguity.
                        List 2–3 plausible paths + implications.
                        Ask user.
```

Two modes, one discipline:

| | Catches at | Prevents |
|--|-----------|----------|
| **Action-routing** | Output boundary | Wrong side-effect fires |
| **Decision-routing** | Planning/design step | Wrong path chosen silently, then built upon |

Most expensive agent failures can be prevented by one of these two. Skipping both is how you get confidently-wrong output delivered without friction.

## Connection to the existing skill

The current `workflows` skill is essentially a **control pattern**: sequential steps with quality checks between them. That's one shape among many. The rewrite should:
- Position sequential workflow as one option, not the default.
- Surface ralph loops, parallel orchestration, and dispatch as siblings.
- Note that control choice is downstream of memory/context choices — you can't ralph-loop if context blows up after 3 iterations.
- Treat **confidence-routed branching** as the default for write actions where mistakes have real cost.
