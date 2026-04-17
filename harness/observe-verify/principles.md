# Observe & Verify

Closing the loop: did the action actually do what was intended?

## Base verification signals

- **Browser screenshots** — visual confirmation of UI state
- **Test results** — pass/fail signals from suites
- **Logs** — runtime trace, error output

These feed back into context so the next turn can react.

## Why this is its own quadrant (not just another action)

Action is "do the thing." Observe-verify is "confirm the thing happened correctly." Conflating them is a common workflow bug — the agent moves on before checking, accumulates errors silently.

The existing SKILL.md already says "Separate steps that generate from steps that evaluate — a step should not judge its own output." That's the workflow-level expression of this principle.

## Patterns

- **After every action** (high cost, high reliability)
- **At natural milestones** (lower cost, slower error detection)
- **Only on failure suspicion** (cheapest, requires the model to know when to look)

## Proactive review: the five validation dimensions

Stripped of role ceremony (CEO / eng / design / DX / CSO), a complete review of a workflow tests five dimensions. They apply both before launch (proactive audit) and after a failure (reactive diagnostic: *which dimension failed?*).

| Dimension | Asks | Failure signature |
|-----------|------|-------------------|
| **Strategic fit** | Does this serve the actual goal, or a surface-level version? | Works as specified, but the specification was wrong |
| **UX** | Latency, clarity, wait states, escalation, human-in-the-loop feel? | User gives up or routes around the workflow |
| **Technical correctness** | Does the logic hold? Are edge cases covered? | Wrong output under conditions the design missed |
| **Maintainability** | Can someone else reason about, debug, or extend this? | Works today, un-fixable tomorrow |
| **Safety / risk** | What can go wrong, and how bad is the blast radius? | Rare failure with high cost — undetected until it fires |

**Maintainability deserves emphasis for agent workflows.** Unlike traditional code, an agent workflow rots silently: skill-file sprawl, opaque memory decisions, prompt tweaks nobody remembers making. Maintainability tests:

- Can a new person load context by reading 2–3 files?
- Is every skill file's purpose obvious from its description?
- Are memory choices explained (why iterative summary? why this a hydrated loop?)
- Is the mutable / immutable boundary written down?

If any answer is no, the workflow will stall the first time someone other than the author tries to change it.

## Context-degradation failure modes

A workflow rarely fails because the model is "wrong." It fails because the context it was given was degraded in one of five concrete ways. These are the diagnostic categories for any post-mortem:

| Mode | What happens | Signature |
|------|--------------|-----------|
| **Lost-in-middle** | Information placed in the middle of a long context suffers 10–40% recall drop due to U-shaped attention | Model ignores a fact that's demonstrably in context; happens when prompt is long |
| **Context poisoning** | Unverified tool output (or a prior hallucination) gets self-referenced and compounds | Later turns inherit a false premise from earlier ones; gets more wrong over time |
| **Distraction** | Even one irrelevant document measurably degrades performance on the relevant task | Adding context makes answer worse, not better |
| **Confusion** | Multiple task types in one context cause model to apply wrong-domain rules | Model treats this task like a sibling task; constraints bleed across boundaries |
| **Clash** | Two individually-correct sources contradict; resolution is unpredictable | Output arbitrarily picks one side, or splits the difference incoherently |

Detection signals worth instrumenting:
- **Performance cliff edges** — quality drops are non-linear. Watch for discontinuities, not gradual decline.
- **Context length vs. quality** — pair every quality metric with the context size at evaluation time.
- **Retrieved-document validation** — fail fast on contradictory or suspect retrievals rather than injecting them.
- **Length ablation** — run identical prompts at varying context sizes to isolate length as the root cause.

Four strategic responses (pick one per failure): **write** (put the critical info at the edges, not middle), **select** (cut the distraction), **compress** (summarize the middle), **isolate** (split tasks across separate contexts).
