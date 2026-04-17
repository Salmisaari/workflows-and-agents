# Runtime Sizing

The architectural argument for a small runtime plus content-rich skill files. Sits in apparent tension with "own your harness / own your memory" but resolves cleanly (see `../architecture.md`).

## The thesis

> "Lean runtime, content-rich skills."

The runtime does four things only:
1. Run the model in a loop
2. Read/write files
3. Manage context (compaction, injection)
4. Enforce safety

This typically comes in at ~200 LOC. Everything else — domain knowledge, process, judgment — lives in markdown skill files outside the runtime.

## The anti-pattern: bloated runtime, shallow skills

Symptoms:
- 40+ tool definitions registered, eating half the context window
- "God-tools" via MCP with 2–5 second round-trips per call
- REST API wrappers that turn every endpoint into a separate tool
- "3x the tokens, 3x the latency, 3x the failure rate"

Concrete contrast:
- **Bad:** Chrome MCP doing screenshot → find → click → wait → read = 15 seconds.
- **Good:** Playwright CLI doing each step in 100ms = 75x faster.

The lesson is more general: **software doesn't have to be precious anymore**. Build exactly the narrow, fast tool you need. Don't reach for the kitchen-sink integration when a 50-line CLI does it.

## Why content-rich skills win

Skills (markdown) are:
- **Permanent.** They don't degrade. They don't forget.
- **Compounding.** Each new skill is a permanent upgrade.
- **Auto-improving.** When the model gets better, every skill instantly gets better — judgment in the latent steps improves while the deterministic steps stay reliable.
- **Portable.** Just files. Move them between harnesses, providers, models.
- **Auditable.** You can read them. Diff them. Version them.

Compare to alternatives where logic is buried in Python wrappers, MCP server code, or proprietary platforms.

## How this reconciles with "own your harness, own your memory"

The principle: **own your harness so you own your memory.** A lean runtime + content-rich skills is one concrete way to satisfy it:
- The 200-line runtime is cheap to own and easy to read.
- Skills are markdown files in a directory you control.
- No long-term memory ever leaves your filesystem.
- Model swap = literally change the API endpoint, skills work unchanged.

These framings aren't in conflict — lean runtime + content-rich skills is what an open harness looks like in practice.

## Implications for the existing SKILL.md

The current `workflows` skill leans toward runtime-bloated thinking — it tries to encode every consideration in one place (preferences, workflow design, layers before generation, tool design, step contracts, review stages, interaction protocol). That's too much runtime, not enough skill.

A leaner version would:
- Be much shorter (the discipline is the value, not the comprehensiveness).
- Push details into linked sub-skills or referenced docs (the "resolver" pattern — see `../resolvers/principles.md`).
- Trust the model to apply judgment instead of forcing rigid stage gates.
