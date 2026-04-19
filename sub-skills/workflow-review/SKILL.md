---
name: workflow-review
description: Use when auditing or debugging an agentic workflow. Two modes — proactive audit (five dimensions — strategic fit, UX, technical correctness, maintainability, safety) and reactive debug (five context-degradation modes — lost-in-middle, poisoning, distraction, confusion, clash). Triggers on "review this workflow", "audit my agent", "why is my agent going wrong", "post-mortem on this run", "check before ship".
---

# Workflow Review

Two entry points, one diagnostic spine. **Audit mode** runs before launch — does the design hold across all five dimensions? **Debug mode** runs after a failure — which degradation mode produced this output? Both end with the same question: *which dimension failed, and what is the targeted response?*

The discipline is to refuse the comfortable answer ("the model is just bad at this") in favour of the specific one ("context poisoning at turn 4 — the agent inherited a hallucinated price and never re-verified"). Specific answers point to fixes; comfortable ones don't.

## When to run this skill

- **Before ship** — run audit mode. The cost is one design session; the saving is the failure mode you would otherwise discover in production.
- **After any failure that surprised you** — run debug mode. Every "the agent went wrong" incident has a named cause; finding it is faster with the framework than without.
- **At a periodic checkpoint** — run audit mode again after every ~100 production runs or any major scope change. Maintainability is the dimension that decays silently between scheduled reviews.

## Mode A — proactive audit

Walk all five dimensions in order. Each has a question, a failure signature, and a concrete test you can run against the workflow as it stands.

### A.1 Strategic fit

*Does this workflow serve the actual goal, or only the surface-level version of it?*

Failure signature: works as specified, but the specification was wrong. The agent perfectly executes a returns flow, and the user wanted to talk to a human. The agent perfectly summarises the document, and the user wanted to act on it.

Test: state the user's underlying job-to-be-done in one sentence, then trace it through the scenario tree from `scenarios`. Are the in-scope branches the ones the user actually arrives with? If the in-scope branches and the real-world inputs diverge, strategic fit is the failure waiting to happen.

### A.2 UX

*Does the interaction surface respect the user's attention, latency tolerance, and authority?*

Failure signature: users give up, route around the workflow, or treat its outputs as advisory rather than acting on them.

Test: read the UX Contract from `workflow-ux`. For each output type, can you answer (a) what does the user see during the wait, (b) which confidence band routes here, (c) what is the escape hatch when they disagree? Any "we'll figure it out at implementation time" answer is a UX failure already shipped.

### A.3 Technical correctness

*Does the logic hold under the inputs the design committed to handling?*

Failure signature: wrong output under conditions the design missed — usually a branch from the scenario tree that was marked in-scope but never actually exercised.

Test: pick five branches from the in-scope list at random and walk them by hand against the current implementation. The randomness matters — humans test the cases they remember, which are the ones that already work.

### A.4 Maintainability

*Can someone other than the author reason about, debug, or extend this workflow six months from now?*

Failure signature: works today, un-fixable tomorrow. Skill-file sprawl, opaque memory choices, prompt tweaks that nobody remembers making.

Tests, all of which must answer yes:

- Can a new person load full context by reading 2–3 files?
- Is every skill file's purpose obvious from its description in under 10 seconds?
- Are memory choices (compaction layers, what's pinned, what's queryable) explained, not just configured?
- Is the mutable / immutable boundary from `constraints` written down?

Maintainability is the silently-decaying dimension. If audit mode skips it, the workflow rots between audits.

### A.5 Safety / risk

*What can go wrong, and how bad is the blast radius when it does?*

Failure signature: rare failure with high cost, undetected until it fires for the first time. The destructive action that should have been in `draft` ran in `auto`. The deny list missed a tool category.

Test: read the Constraints Document and the UX Contract together. For every action the workflow can take, name (a) reversibility, (b) which permission mode allows it, (c) which deny rule would block it. Anything that lands in "irreversible + auto + no deny rule" is a safety failure waiting to ship.

## Mode B — reactive debug

When the workflow has produced a wrong or surprising output, walk these six failure classes in order. Stop at the first match — usually one explains most of the symptom.

### B.1 Lost-in-middle

*Did the model demonstrably ignore information that was in context?*

Signature: the relevant fact was provably present, but the answer behaved as though it weren't. Worse on long prompts.

Test: extract the prompt. Is the critical information in the middle of a long context? If yes, this is the failure. Response: **write** — move the critical info to the edges (top of system prompt or end of user message).

### B.2 Context poisoning

*Did an unverified earlier output get treated as fact in a later turn?*

Signature: later turns inherit a false premise from earlier ones, and the wrongness compounds.

Test: walk the trace backward from the wrong output. Is there an unverified tool result, a hallucination, or an assumption that was never checked? Response: **select** — drop the poisoned turn from context, re-run with verification gates (observe-verify after the suspect step).

### B.3 Distraction

*Was an irrelevant document or section degrading the answer?*

Signature: adding context made answers worse, not better. Often a "we just included everything we had" pattern.

Test: re-run with the irrelevant section removed. Does quality recover? Response: **select** — cut the distractor from context. If you can't tell what's relevant ahead of time, build a router/retriever instead of bulk-injecting.

### B.4 Confusion

*Were multiple task types mixed in one context?*

Signature: rules from one domain bled into another. A coding task got customer-service formatting. A summary inherited a search-tool persona.

Test: count the distinct tasks in the prompt. If more than one, this is the suspect. Response: **isolate** — split the tasks across separate contexts (sub-agents, sequential calls, or separate sessions).

### B.5 Clash

*Did two individually-correct sources contradict, with the model picking arbitrarily?*

Signature: output picks one source at random or splits the difference incoherently. Often only visible across multiple runs.

Test: scan the prompt for two sources giving conflicting facts. Response: **compress** at the source — resolve the clash deterministically before the model sees it (pick one, version-stamp, or flag the contradiction explicitly).

### B.6 Stop-at-ambiguity (the sibling failure)

*Was there a decision point the agent should have stopped at but didn't?*

Signature: the agent confidently went the wrong way at a fork, then built ten minutes of work on top. Distinct from the five context modes — the context was fine; the runtime discipline was missing.

Test: walk the trace forward to the first wrong decision. Could the agent have known it had two plausible paths? If yes, this is the failure. Response: add explicit confidence calibration and a stop-condition for unclear judgment to the system prompt; route low-confidence forks to `clarify` per the UX Contract.

## Step C — pick the response

Each context-degradation mode has a default strategic response. Don't combine them indiscriminately:

| Failure | Response | Mechanism |
|---------|----------|-----------|
| Lost-in-middle | **write** | Reposition critical info to context edges |
| Context poisoning | **select** | Drop poisoned turns; add verification |
| Distraction | **select** | Cut irrelevant context |
| Confusion | **isolate** | Split tasks across sub-agents / sessions |
| Clash | **compress** | Resolve conflict before the model sees it |
| Ambiguity | (runtime) | Add stop-condition + confidence routing |

Picking two responses for one failure is usually overcorrection. Pick the right one, ship, re-measure.

## Output contract

The skill produces a **Review Report** with five parts:

1. **Mode.** Audit or debug, with the trigger (pre-ship checkpoint, post-incident, scheduled).
2. **Findings per dimension** (audit) or **failure mode identified** (debug). Specific, with evidence — not "the agent is bad at X" but "branch B.3 from scenario tree returns wrong output when SKU is missing."
3. **Severity classification.** Per finding: blocker / serious / minor / cosmetic.
4. **Targeted responses.** What to change, where, and which response category (write / select / compress / isolate / runtime / scope).
5. **Re-test plan.** How you will verify the fix worked — and what would tell you it didn't.

Hand the report back to whichever skill produced the broken artifact (`scenarios` if scope was wrong, `constraints` if the boundary leaked, `memory-system` if compaction lost something, `workflow-ux` if interaction failed). Review is not a terminal step — it routes back into the design loop.

## Failure modes

- **Stopping at "the model is bad at this."** This is never a finding. The failure has a named class; do the work to find it.
- **Skipping maintainability.** The dimension users never complain about until the original author leaves. Audit it on every pass.
- **Combining responses.** Applying write + select + compress to the same failure obscures what actually fixed it. Pick one, measure, iterate.
- **Treating debug mode as one-shot.** A single post-mortem captures one incident. Patterns across incidents are where the architectural fixes live — keep the report archive scannable.
- **No re-test.** A fix without a re-test is a hope. The re-test plan is what turns review into a closed loop.

Back-references: `../../harness/observe-verify/principles.md` (five validation dimensions, five context-degradation modes with the write/select/compress/isolate responses); `../../architecture.md` (stop-at-ambiguity as the sibling failure class).
