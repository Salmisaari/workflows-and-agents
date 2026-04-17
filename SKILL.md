---
name: design-workflow
description: Use when designing a new agentic workflow from scratch — "how should I build an agent that…", "design a workflow for…", "what's the right shape for…". Runs the diagnostic, picks the execution pattern, routes to specialized skills (memory-system, scenarios, constraints, workflow-review, workflow-ux) as the question narrows.
---

# Design Workflow

Entry point for designing a new agentic workflow. Run the diagnostic first. Work the sequence in order. Route to a specialized skill the moment the question narrows.

## Diagnostic

Answer four questions before proposing any architecture. If two answers conflict, stop and surface both options to the user.

**Q1 — Is a user waiting for output?**
Yes → interactive shape (REPL/ReAct, hydrated loop, full loop).
No → background shape (pipeline, state machine, scheduled digest).

**Q2 — Are the steps deterministic or latent?**
All deterministic → pipeline or state machine.
Any step needs latent reasoning → agent loop (hydrated or full).

**Q3 — Terminal or continuous?**
Terminal → pipeline / one-shot.
Continuous, polling, scheduled → state machine or scheduled digest.

**Q4 — Tools, and how many turns?**
No tools → pipeline / digest.
Tools, bounded turns → hydrated agent loop.
Tools, unbounded turns → full agent loop.

## Pattern catalog

Six recurring shapes. Pick one. Do not invent a seventh.

- **Pipeline** — linear sequence of deterministic steps. One input, one output, no branching.
- **State machine** — explicit states with typed transitions. Agent moves between states on signals.
- **Scheduled digest** — periodic batch. Ingest → synthesize → write artifact.
- **REPL / ReAct** — interactive loop with user in the loop. Reason, act, observe, repeat.
- **Hydrated agent loop** — entity context pre-loaded. Agent operates on a known subject until done.
- **Full agent loop** — unbounded exploration. Agent decides when to stop.

Picking the wrong shape is the top cause of fragile agentic systems. If torn between two, list both with their tradeoffs and ask the user before committing.

## Design sequence

Once the pattern is chosen, work these steps in order. Skipping earlier steps leaks into later ones.

1. **Enumerate scenarios.** Do not design from the happy path. Invoke `scenarios` to list every realistic input the workflow will receive — including malformed, adversarial, and edge cases.
2. **Define constraints.** Before writing any behavior rules, state what the agent can change and what it cannot. Invoke `constraints`. Ground truth and evaluation must live in the immutable zone.
3. **Design the memory architecture.** Invoke `memory-system`. Answer the seven memory questions before committing to a shape.
4. **Design the interaction surface.** If a human is in the loop, invoke `workflow-ux`. Cover state handling while waiting, latency disclosure, reminders, and escalation paths.
5. **Review before ship.** Invoke `workflow-review` and validate against all five dimensions (strategic fit, UX, technical correctness, maintainability, safety).

## Design DNA

Hold these four rules while designing:

- **Think before acting.** State assumptions explicitly. If two interpretations exist, surface both. Do not pick silently.
- **Simplicity first.** Minimum structure that solves the problem. No speculative abstractions. No configurability not asked for.
- **Surgical changes.** Touch only what the design requires. Do not refactor adjacent concerns.
- **Goal-driven.** Define success criteria before sketching architecture. "Make it work" is not a criterion.

## Stop at ambiguity

The most expensive failure mode in agentic systems is confidently picking the wrong path at an ambiguous decision and building on it. Prevent it:

- When uncertain, list two or three plausible alternatives with their tradeoffs.
- Name the exact thing that is ambiguous — a requirement, a constraint, a user intent, a pattern fit.
- Surface the choice to the user before continuing.
- Do not rationalize a single path forward. A missed branch point costs more later than a one-sentence question costs now.

Apply this discipline at two levels: the design session itself, AND the runtime behavior of the workflow being designed. Build stop-at-ambiguity into the agent's decision loop, not just into how this skill operates.

## When to stop using this skill

- Question narrows to one sub-topic → route to the matching skill and let this one drop out of context.
- User asks for code or prompt-level implementation → this skill is for architecture only; defer to the appropriate coding context.
- Design is approved → stop advising and return control.
