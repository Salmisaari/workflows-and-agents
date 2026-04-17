# Skill Principles

Skills are the content-rich layer in the lean-runtime architecture. They encode the judgment, process, and domain knowledge that turns a generic model into a specialist.

## What a skill is

A reusable markdown document that teaches the model **how** to do something. The user supplies **what**.

> "The skill describes a process of judgment. The invocation supplies the world."

## Skill-as-method-call

The defining insight: a skill takes **parameters**. Same procedure, different arguments → different capability.

Example: `/investigate` with seven steps (scope → timeline → diarize → synthesize → argue both sides → cite). Parameters: `TARGET`, `QUESTION`, `DATASET`.

| Invocation | Result |
|------------|--------|
| TARGET = safety scientist, DATASET = 2.1M discovery emails | Medical research analyst investigating whistleblower silencing |
| TARGET = shell company, DATASET = FEC filings | Forensic investigator tracing coordinated donations |

Same skill. Same seven steps. Same markdown file.

This is **software design with markdown as the programming language and human judgment as the runtime.** Markdown is a *more perfect* encapsulation than rigid source code for capabilities that involve judgment, because it describes process in the language the model already thinks in.

## The "no one-off work" rule

The rule, worth keeping in any rewritten skill:

> "You are not allowed to do one-off work. If I ask you to do something and it's the kind of thing that will need to happen again, you must: do it manually the first time on 3 to 10 items. Show me the output. If I approve, codify it into a skill file. If it should run automatically, put it on a cron.
>
> The test: if I have to ask you for something twice, you failed."

The discipline:
1. Manual on 3–10 items first (proves the procedure).
2. Show output for approval.
3. Codify into a skill file.
4. Optionally schedule on cron.

## Skills are permanent upgrades

A working skill never degrades. It compounds. When the next model drops, every skill gets better automatically — the judgment steps improve, the deterministic steps stay reliable.

This is the mechanism behind the 100x productivity gap. Not a smarter model. The discipline of codifying every repeatable judgment as a skill.

## The learning loop (skills that rewrite themselves)

After execution, a feedback skill reads outcomes (NPS, errors, "OK" responses), diarizes the mediocre ones, extracts patterns, and **writes new rules back into the relevant skill files.**

Example pattern:
```
When attendee says "AI infrastructure"
  but startup is 80%+ billing code:
  → Classify as FinTech, not AI Infra.

When two attendees in same group already know each other:
  → Penalize proximity.
     Prioritize novel introductions.
```

These rules live in the skill file. The next run uses them automatically. Quality metrics move measurably across subsequent runs. **The skill file learned what "OK" actually meant.**

The general loop transferable across knowledge work:
- **Forward path:** retrieve → read → diarize → count → synthesize.
- **Improvement path:** survey → investigate → diarize mediocre cases → rewrite the skill.

## Diarization (the synthesis step)

The step that makes AI useful for real knowledge work. The model reads everything about a subject and writes a structured profile — a single page of judgment distilled from dozens or hundreds of documents.

Critical property: **no SQL query, no RAG pipeline, no embedding search produces this.** The model has to actually read, hold contradictions in mind, notice what changed and when, synthesize.

Example output shape:
```
FOUNDER: Maria Santos
COMPANY: Contrail (contrail.dev)
SAYS: "Datadog for AI agents"
ACTUALLY BUILDING: 80% of commits are in billing module.
  She's building a FinOps tool disguised as observability.
```

That `SAYS` vs `ACTUALLY BUILDING` gap is unfindable by keyword or embedding similarity. It requires reading the application + GitHub history + advisor transcripts and judging the gap.

This is the canonical example of work that **must be in latent space** — and skills that lean on diarization unlock value no traditional pipeline can.

## Skill encoding spectrum

Real-world systems encode "skills" in three meaningfully different ways. These are not interchangeable — choosing the wrong one gives the wrong ergonomics.

| Pattern | Skill is... | Invocation | Parameters | Best for |
|---------|-------------|------------|------------|----------|
| **Method-call markdown** | A procedure called by name with arguments | Explicit: `/investigate TARGET=x QUESTION=y` | Yes, named | Repeatable workflows where the user supplies the world |
| **Declarative reference material** | Guidance the model reads and decides when to apply | Implicit: model matches situation to skill description | No | Ambient capabilities, "if you see X, do Y" patterns |
| **Branch + interactive setup** | A code change + setup walkthrough | Install-time, then ambient | At setup time, not per-call | Capabilities that need code installed (integrations, CLI tools) |

A single agent system can use all three. The deciding question:
- *Will this be called many times with different inputs?* → method-call markdown
- *Should this always be available as background guidance?* → declarative reference
- *Does this require code/dependencies to be installed?* → branch + setup

The current `workflows` skill treats skills as one shape (procedural). The rewrite should at minimum acknowledge the spectrum so users pick the right encoding.

## Skill metadata as resolver bait

In a system with many skills, the **description field is load-bearing infrastructure** (see `../resolvers/principles.md`). It's how the harness routes user intent to skill. Practical implications:

- Description must contain the **trigger words** users actually type, not just abstract category names
- Description should distinguish from sibling skills (no two should match the same intent)
- Description is part of the always-on context budget — every active skill costs tokens on every turn

Conditional activation metadata (e.g., `platforms`, `requires`, `fallback_for`) lets a skill express "I'm only relevant when X" — keeps the description list lean.

## Skills that learn vs skills that don't

Two postures:
- **Static skill** — written once, edited only by humans; predictable, auditable, no surprises
- **Self-modifying skill** — feedback loop writes new rules back into the skill file

Self-modification is powerful (the "improvement loop" — survey → diarize mediocre cases → rewrite the skill) but introduces risks:
- **Rule rot** — old rules contradict new ones; no garbage collection
- **Drift** — skill gradually moves away from author's intent
- **Audit complexity** — what was the skill at the moment of any given decision?

Practical guidance: self-modifying skills need versioning + a human review checkpoint before new rules go live, or they need a clear scope boundary (e.g., only the `# Learned Rules` section is auto-edited).

## Skill design checklist (provisional)

For each skill, decide:
1. **Single sentence of purpose.** If you need two sentences, it's two skills.
2. **Parameters.** What does the invocation supply? Name them clearly (TARGET, QUESTION, DATASET pattern).
3. **Steps.** Numbered. Each step is one of: latent (model judgment) or deterministic (call a tool).
4. **Latent / deterministic split.** Mark every step. Move work to the right side.
5. **Output contract.** What downstream skills (or humans) can rely on receiving.
6. **Failure / escalation.** What does this skill do when it can't proceed?
7. **Learning hook.** Where does feedback land? Which lines get rewritten when patterns emerge?
