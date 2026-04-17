# Skills

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

Same skill, same seven steps, same markdown file.

This is **software design with markdown as the programming language and human judgment as the runtime** — a more perfect encapsulation than rigid code for capabilities that involve judgment, because it describes process in the language the model already thinks in.

## The "no one-off work" rule

The discipline:

> "You are not allowed to do one-off work. If I ask you to do something and it's the kind of thing that will need to happen again, you must: do it manually the first time on 3 to 10 items. Show me the output. If I approve, codify it into a skill file. If it should run automatically, put it on a cron.
>
> The test: if I have to ask you for something twice, you failed."

Four steps:

1. Manual on 3–10 items first — proves the procedure.
2. Show output for approval.
3. Codify into a skill file.
4. Optionally schedule on cron.

## Skills are permanent upgrades

A working skill never degrades. It compounds. When the next model drops, every skill gets better automatically — judgment steps improve, deterministic steps stay reliable. And because skills are plain markdown files, they're portable and auditable in a way code-encoded logic isn't: you can read them, diff them, version them, and move them between harnesses, providers, or models without rewriting. Compare that to alternatives where logic is buried in Python wrappers, MCP server code, or proprietary platforms — those rot when the model or the provider changes.

This is the mechanism behind the 100× productivity gap. Not a smarter model. The discipline of codifying every repeatable judgment as a skill — in a form that survives model, provider, and harness changes.

## The learning loop

After execution, a feedback skill reads outcomes (NPS, errors, "OK" responses), diarizes the mediocre ones, extracts patterns, and **writes new rules back into the relevant skill files.**

Example patterns the loop might produce:

```
When attendee says "AI infrastructure"
  but startup is 80%+ billing code:
  → Classify as FinTech, not AI Infra.

When two attendees in same group already know each other:
  → Penalize proximity.
     Prioritize novel introductions.
```

These rules live in the skill file. The next run uses them automatically. Quality metrics move measurably across subsequent runs. **The skill file learned what "OK" actually meant.**

The general loop, transferable across knowledge work:

- **Forward path:** retrieve → read → diarize → count → synthesize.
- **Improvement path:** survey → investigate → diarize mediocre cases → rewrite the skill.

## Diarization

The step that makes AI useful for real knowledge work. The model reads everything about a subject and writes a structured profile — a single page of judgment distilled from dozens or hundreds of documents.

Critical property: **no SQL query, no RAG pipeline, no embedding search produces this.** The model has to actually read, hold contradictions in mind, notice what changed and when, synthesize.

Example output:

```
FOUNDER: Maria Santos
COMPANY: Contrail (contrail.dev)
SAYS: "Datadog for AI agents"
ACTUALLY BUILDING: 80% of commits are in billing module.
  She's building a FinOps tool disguised as observability.
```

The `SAYS` vs. `ACTUALLY BUILDING` gap is unfindable by keyword or embedding similarity. It requires reading the application + GitHub history + advisor transcripts, then judging the gap.

This is the canonical example of work that **must be in latent space**. Skills that lean on diarization unlock value no traditional pipeline can.

## Skill encoding spectrum

Real-world systems encode "skills" in three meaningfully different ways. Choosing the wrong one gives the wrong ergonomics:

| Pattern | Skill is... | Invocation | Parameters | Best for |
|---------|-------------|------------|------------|----------|
| **Method-call markdown** | A procedure called by name with arguments | Explicit: `/investigate TARGET=x QUESTION=y` | Yes, named | Repeatable workflows where the user supplies the world |
| **Declarative reference** | Guidance the model reads and applies when relevant | Implicit: model matches situation to description | No | Ambient capabilities, "if you see X, do Y" patterns |
| **Branch + interactive setup** | A code change + setup walkthrough | Install-time, then ambient | At setup time, not per-call | Capabilities that need code installed (integrations, CLI tools) |

A single system can use all three. The deciding questions:

- Called many times with different inputs? → method-call markdown.
- Always-available background guidance? → declarative reference.
- Needs code or dependencies installed? → branch + setup.

Treating all skills as one shape (usually procedural) collapses the spectrum. Naming the three lets the author pick the right one per capability.

## Skill metadata as resolver bait

In a system with many skills, the **description field is load-bearing infrastructure** (see `../resolvers/principles.md`). It's how the harness routes user intent to the right skill.

- Descriptions must contain the **trigger words** users actually type, not abstract category names.
- Descriptions must distinguish the skill from siblings — no two should match the same intent.
- Descriptions sit in the always-on context budget. Every active skill costs tokens on every turn.

Conditional activation metadata (`platforms`, `requires`, `fallback_for`) lets a skill express "I'm only relevant when X" — keeps the description list lean.

## Skills that learn vs. skills that don't

Two postures:

- **Static skill** — written once, edited only by humans. Predictable, auditable, no surprises.
- **Self-modifying skill** — the feedback loop writes new rules back into the skill file.

Self-modification is powerful but introduces real risks: **rule rot** (old rules contradict new ones with no garbage collection), **drift** (the skill slowly moves away from author intent), **audit complexity** (what was the skill at the moment of any given decision?).

Practical guidance: self-modifying skills need versioning and a human review checkpoint before new rules go live, or a clear scope boundary (e.g., only the `# Learned Rules` section is auto-edited).

## Skill design checklist

For each new skill, decide:

1. **Single sentence of purpose.** If you need two sentences, it's two skills.
2. **Parameters.** What does the invocation supply? Name them clearly (TARGET, QUESTION, DATASET pattern).
3. **Steps.** Numbered. Each step is either latent (model judgment) or deterministic (tool call).
4. **Latent / deterministic split.** Mark every step. Move work to the right side.
5. **Output contract.** What downstream skills (or humans) can rely on receiving.
6. **Failure / escalation.** What does this skill do when it can't proceed?
7. **Learning hook.** Where does feedback land? Which lines get rewritten when patterns emerge?
