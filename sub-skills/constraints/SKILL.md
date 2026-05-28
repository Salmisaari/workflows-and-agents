---
name: constraints
description: Use when defining what an agent can and cannot change — the mutable / immutable boundary. Triggers on "what can the agent modify", "define constraints", "set the agent's autonomy boundary", "where does ground truth live". Ensures evaluation harness and ground truth sit in the immutable zone so the agent can't improve-away its own checks.
---

# Constraints

The agent's autonomy is defined by what it *cannot* change. This skill draws that boundary explicitly. The output is a constraints document the runtime can enforce and the agent can read.

The principle is counterintuitive: **constraints enable autonomy.** An agent that could rewrite its evaluation harness can always claim improvement. An agent that could rewrite the data pipeline produces incomparable runs. Without an immutable floor, every result becomes negotiable, and the workflow loses its grip on truth.

## When to run this skill

Run after `scenarios` and before any implementation work. The in-scope branches from the scenario document tell you *what* the agent must do; this skill defines the surface it does it on. Re-run whenever scope shifts or a new agent capability is added.

## Step 1: Name the artifact

What is the agent producing or modifying? Be specific. A code agent produces a file or directory; a research agent produces a report or a training run; a customer-ops agent produces a draft response. The artifact is the *only* thing the agent should think of as mutable in the working scope.

A workflow with no clearly named artifact has no clearly named scope. If you cannot name the artifact in one sentence, return to `scenarios`.

## Step 2: Identify ground truth

Ground truth is the thing the artifact is judged against. Tests, evaluation rubrics, golden outputs, success criteria, business invariants. Whatever it is, it must be **outside** the artifact and the agent's reach.

If the agent can edit the test, the test is not a test. If the agent can edit the rubric, the rubric is not a rubric.

## Step 3: List the immutable scaffolding

Beyond ground truth, name everything else the agent must not modify. The recurring items:

- **Tool definitions and schemas.** The agent does not get to redefine its own tools.
- **Data pipelines and prep logic** (the `prepare.py` of any system). The pipeline produces the input the agent reasons over; if the agent can rewrite it, runs become incomparable.
- **Scope boundary itself.** The list of in-scope and out-of-scope branches from `scenarios`.
- **Skill bodies in production runs.** The agent may invoke skills, but should not edit them at runtime — edits are a human-reviewed change.
- **Audit logs and persistence directories.** The agent writes new entries; it does not rewrite history.

## Step 4: Worked example

A research agent has full freedom inside `train.py` — model architecture, hyperparameters, training loop, anything. It cannot modify:

- `prepare.py` (the data pipeline)
- The evaluation harness
- The dependency graph

| Mutable (agent's playground) | Immutable (the floor it stands on) |
|------------------------------|------------------------------------|
| `train.py`: model code, hyperparameters | `prepare.py`: data pipeline |
| Run notes, scratchpad | Eval harness, golden metrics |
| Choice of approach within scope | Scope boundary itself |
| Skill invocation order | Skill bodies (production) |

The agent improves the model freely. The evaluation question — *"did this run actually beat baseline?"* — stays answerable because the harness is fixed.

## Step 5: Make the constraints visible to the agent

The system prompt must contain the constraints. An agent operating against an invisible fence is a failure mode: it spends turns trying to do things it isn't allowed to do, then either succeeds (a security incident) or fails confusingly.

Format the constraints as a short block in the system prompt:

```
You can modify: <artifact>
You cannot modify: <list of immutable surfaces>
If you find yourself wanting to modify something on the cannot-modify list,
stop and surface the conflict to the user.
```

The last line matters: an immutable surface is a signal that the *task* may be wrong, not just the agent's plan. Stopping at the boundary is correct behavior, not an apology.

## Step 6: Set the human-driven revision process

Constraints are not eternal — scopes shift, new tools land, eval criteria evolve. But constraints must change through a **human-driven process**, not by the agent loosening its own boundaries. Define the channel: a PR review, a design doc update, a checkpoint conversation. Whatever it is, the agent does not initiate it; the agent surfaces the need, the human decides.

Without this discipline, scope creep becomes silent — the agent quietly stretches its surface over many runs, and the immutable floor erodes.

## Output contract

The skill produces a **Constraints Document** with four parts:

1. **Artifact.** One-sentence name of what the agent produces.
2. **Mutable list.** Everything the agent may freely change.
3. **Immutable list.** Everything the agent must not touch, with one-line rationale per item.
4. **Revision process.** How an immutable surface gets changed, and by whom.

This document feeds the system prompt (Step 5) and is one of the inputs to `workflow-review` later, where it is checked under the "safety / risk" dimension.

## Constraints within skill files

The mutable/immutable boundary also applies *inside* the artifact when the artifact is subject to iterative self-improvement — a skill file being optimized, a prompt being refined, a plan being revised across cycles.

The pattern: split the file into a **protected section** (slow-won lessons that change only through a broadly-sampled, gated process) and a **working section** (hypotheses open to bounded editing each iteration). The protected section is immutable within an optimization pass. This prevents a sequence of locally-correct fast edits from overwriting a globally-important lesson that took many runs to discover.

When defining constraints for a self-improving workflow, add a fifth item to the immutable list:

- **Protected sections of the artifact itself.** If the agent can edit the slow-lessons section during fast iteration, the optimization loop loses its memory. Mark the boundary structurally (a named section, a frontmatter field) so it's visible to every editor — human or agent.

## Failure modes

- **Ground truth in the mutable zone.** If tests, rubrics, or eval data live where the agent can edit them, the workflow has no real check. Fix immediately — this is the most expensive constraint failure.
- **Implicit constraints.** Constraints the human knows but the agent does not are constraints that exist only after a violation.
- **Agent-driven boundary changes.** The agent must never silently expand what it can touch. Any expansion is a human action.

Back-references: `../../architecture.md` "Constraints principle" — the underlying principle, the worked example, the mutable/immutable table. See also "The self-improving loop" and `../self-improving-loop/SKILL.md`, where this read-only wall on evidence (ground truth and traces mounted immutable) is the load-bearing invariant.
