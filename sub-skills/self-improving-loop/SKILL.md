---
name: self-improving-loop
description: Use when designing a workflow that should get measurably better at a bounded task over time by turning the corrections experts already make into evals that drive validated, human-gated changes. Triggers on "self-improving agent", "agent that gets better over time", "eval-driven improvement loop", "turn corrections into evals", "the model alone isn't improving". Fits the class where inputs are messy, outputs must be correct, and humans already correct the outputs as part of their job.
---

# Self-Improving Loop

A workflow that gets measurably better at a bounded task over time — not by training the model, but by turning the work it already does into signal that drives validated, human-gated engineering changes. The model is fixed. What evolves is the product around it: extraction schemas, source-selection logic, mappers, prompts, skill files, and above all the eval set. Design it when the model alone has plateaued but experts keep correcting the same kinds of mistakes.

For the theory — why it compounds, the two-loops / two-valves model, the full invariant list — see `../../architecture.md` "The self-improving loop." This skill is how you build one.

## When this pattern fits

Three preconditions must all hold. If any fails, this is the wrong pattern — return to `design-workflow` and pick from the catalog.

1. **Inputs are messy and unstructured.** Notes, emails, PDFs, spreadsheets — no fixed schema.
2. **Outputs must be correct.** There is a right answer and being wrong has a cost — a filed return, a booked invoice, a shipped order.
3. **Experts already correct the outputs as part of their job.** This is the load-bearing one. Those corrections are the free signal. No expert in the loop → no signal → no self-improvement.

The defining property: doing the work generates the improvement signal. Corrections are free labels, so the system compounds instead of plateauing. The proof that it is *improving* and not merely expanding coverage is that accuracy rises *while the task gets harder* — rising accuracy at fixed difficulty is just tuning.

## Key terms

A small vocabulary the rest of the procedure leans on.

- **Trace** — the recorded history of one run: sources → extracted fields (with provenance) → mapped output → final output → correction. The substrate everything downstream reads.
- **Provenance** — a pointer from an extracted value back to the exact source region (file, page, cell). Lets a human approve a value by sight, and lets a failure be *located*, not just observed.
- **Correction diff** — one structured record of a wrong value: `expected · predicted · field_id · source`. The atomic unit of signal.
- **Failure mode** — a *recurring* category of wrongness, distinct from a one-off error.
- **Clustering** — grouping diffs into named failure modes and discarding the noise. The irreplaceable human step.
- **Eval** — an automated test scoring output against expected, with a grader and a pass/fail threshold.
- **Grader** — the code that decides pass/fail. It is product code and can be buggy; "the grader is wrong" is a valid root cause.
- **Hill** — one actionable failure mode packaged as an eval-shaped ticket: `{thesis, source evidence, success metric, eval seed}`. The unit of work the queue holds.
- **Valve** — a point where the human loop hands work to the agent loop and back: the issue queue (in) and the PR (out). Both closed by default.

## Build order

Build the substrate before the loop. Each phase has a done-bar and a guard against running ahead.

**Phase 0 — Trace with provenance.** Make the runtime pipeline emit a structured trace per run, every extracted value carrying a precise source pointer. *Done when:* every run records each field with its source locator. *Do not yet:* build any "learning" — there is nothing to learn from until traces exist.

**Phase 1 — Capture corrections as diffs.** Instrument review so each human correction emits a structured diff against ground truth, rather than a silent re-edit. *Done when:* every correction produces a diff row. *Do not yet:* wire diffs into anything automated.

**Phase 2 — Manual clustering + first eval by hand.** A human clusters a few weeks of diffs into failure modes and *filters the noise*. Hand-build one targeted eval and a small regression suite. *Done when:* one actionable cluster is a runnable eval, and you know roughly what fraction of diffs were noise. *Do not yet:* hand anything to an agent.

**Phase 3 — Issue queue + agent on ONE failure mode.** Stand up the queue with the eval-shaped template. Build the scoped task environment with the read-only wall. Let the agent close one ticket. *Done when:* the agent fixes one failure mode end-to-end and opens a PR with eval evidence, with no human input between ticket and PR. *Do not yet:* scale to many modes or skip the regression suite.

**Phase 4 — PR gate + regression discipline.** Require eval evidence to merge; every shipped eval joins the regression suite. *Done when:* a reviewer can merge in ~30 seconds on evidence alone, and no fix has silently broken a prior fix.

**Phase 5 — Scale failure modes, then domains.** Run tickets in parallel. The first failure mode is expensive — it produces the abstractions (eval conventions, review-row format, environment layout); later ones reuse them and get cheaper. *Done when:* a new failure mode goes cluster → merged fix with no new infrastructure, only new evals.

## The one irreplaceable step

Most correction diffs are *not* failures — they are human preferences, values carried forward from prior runs, and workflow artifacts. A large noise fraction is normal. If raw diffs flow into the loop, the agent optimizes toward noise: it "fixes" preferences as if they were bugs, and the system degrades instead of compounding.

So a human clusters diffs into named failure modes and throws out the noise *before anything is built*. This is the single place where domain judgment is genuinely irreplaceable, and it is what makes the system *self*-improving rather than merely autonomous. Everything else is plumbing; this is the gate. Keep the actionable/noise decision human even after you semi-automate the grouping (group by `field_id` + similarity to surface candidates).

## What the agent may touch

The boundary is the constraints principle applied to this loop (see `../constraints/SKILL.md`). Enforce it with the environment, not by asking the agent nicely.

| Writable (the bounded layer) | Read-only (mounted as evidence) |
|------------------------------|---------------------------------|
| Extraction schema, source selection, mapper, provenance layer | Production traces |
| Evals: datasets, suites, graders | Source artifacts |
| Skills / docs | Ground truth — the approved output |

If the agent can write to the trace, the source, or ground truth, it games the eval by corrupting the evidence. The read-only wall is not optional. The agent does not touch pipeline architecture or output-system contracts.

## Reference contracts

These are the handoff shapes. Adapt field names to your stack; keep the structure.

Correction diff — the atomic signal:
```yaml
run_id: run_2026_0518_0042
field_id: IFDSEGEN.25
expected: "14 days"        # from ground truth
predicted: "0 days"        # what the system produced
source: { uri: owner_note.txt, locator: "line 3" }
disposition: unreviewed    # unreviewed | actionable | noise  (set at clustering)
```

Failure cluster — the clustering output:
```yaml
cluster_id: FIND-RENTAL-0042
thesis: "Leaves fair-rental-days blank when the value is in a free-text note."
disposition: actionable    # actionable -> becomes a ticket; noise -> dropped
n: 7                       # diffs supporting this cluster
candidate_root_cause: "extraction schema has no field for free-text day counts"
```

Hill — the eval-shaped ticket (valve IN). A ticket the agent can run without coming back must carry all four parts:
```markdown
## Thesis
System leaves fair-rental-days blank when the value appears only in free text.
## Source evidence  (N=7)
- run_2026_0518_0042 — owner_note.txt:line 3 — expected "14 days", got "0 days"
## Success metric
>= 90% precision AND recall on the targeted suite, zero regressions.
## Eval seed
targeted: evals/suites/fair-rental-days.yaml · regression: evals/suites/rental-regression.yaml
## Scope
Editable: extraction schema, source selection, mapper, grader. Out: architecture, output contracts.
```

PR — evidence for a merge decision (valve OUT). The reviewer approves proof, not vibes:
```markdown
## Targeted eval  (required)
fair-rental-days: 11/12 pass — precision 0.95, recall 0.92 ✅ (target 0.90)
## Regression  (required)
rental-regression: 148/148 ✅ — no regressions
## Root cause
Schema had no field for free-text day counts; values dropped before mapping.
## Scope check
Edits confined to product/rental-income/. No architecture or contract changes.
```

## Invariants

These are the non-negotiables; violating any one converts the system from self-improving to self-degrading.

- Two human gates, always: a human decides *what is a real failure* (clustering) and *what ships* (merge). Everything between is autonomous.
- Ground truth is immutable; evidence is mounted read-only by the environment, not by instruction.
- Automation is scoped to the bounded layer (extraction + mapping), never architecture or output contracts.
- Evals are versioned files in the repo — datasets, suites, graders. Not notebooks, not vibes.
- The grader is fixable code; "the eval is wrong" is a valid root cause and a valid fix.
- Tickets are eval-shaped; PRs carry targeted + regression results. These are what make autonomy safe and review fast.
- Noise is filtered before it becomes work — raw diffs never reach the agent.

## How you know it's working

Track these, not "does the agent seem smart": completion-threshold curves rising over calendar time; accuracy rising *against rising difficulty* (the signature that distinguishes the loop from mere tuning); ticket cycle time falling as abstractions accumulate; regressions caught pre-merge; reviewer time per PR staying near-constant (~30s) as volume rises. Evals are the observe-verify layer made durable and versioned, with the grader kept separate from the generator (see `../../harness/observe-verify/principles.md`).

## Worked example — Droppe invoice pipeline

A supplier-invoice operation booking invoices into the accounting system.

| Piece | Instantiation |
|-------|----------------|
| Runtime + trace (Phase 0) | OCR → field extraction → mapping to GL codes / line items; each value points back to the exact invoice region. Build first. |
| Correction capture (Phase 1) | Every value a finance agent or human fixes before booking becomes a diff against the booked value. |
| Clustering gate (Phase 2) | Expect a *large* noise fraction — supplier-specific posting preferences and GL conventions are not extraction misses. Filter them ruthlessly. |
| Queue · agent · PR (Phases 3–4) | Eval-shaped tickets; per-ticket branch with the invoice + trace mounted read-only; PRs carry targeted + regression results; a human merges. |
| Feedback (Phase 5) | Merged change deploys → new invoices generate new traces → new clusters → new tickets. |

You do not need both an issue tracker and a code host — you need the two *roles*: a failure queue and a review surface. The one thing that does not come free is the clustering gate, the mechanism that turns a pile of corrections into well-formed, eval-shaped tickets. Everything else is plumbing you mostly have.

## Output contract

Running this skill produces:

1. **A phased build plan** — which of Phases 0–5 exists today, what to build next, and the done-bar for it. Most teams discover they are missing the trace (Phase 0) or the clustering gate (Phase 2).
2. **The read-only wall** — a constraints document (via `../constraints/SKILL.md`) naming the writable bounded layer and the immutable evidence.
3. **The two valve templates** — the hill (issue) template and the PR template, instantiated for the user's domain.

Stop when the team has a concrete next phase with a done-bar. Do not design Phase 5 scaling before Phase 0 traces exist.
