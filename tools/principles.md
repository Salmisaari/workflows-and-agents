# Tools

How the model's decisions become side effects in the world.

## Tool categories

- **Built-in tools** — the runtime's core function set (Read, Edit, Glob, Grep, Write, Bash)
- **MCP tools** — Model Context Protocol servers exposing external tool catalogs

Results flow back into context as part of the next turn's injection.

## What carries forward from existing SKILL.md

Tool design checklist already captured (WHAT / WHEN / INPUTS / RETURNS / MASKING). Worth keeping intact:
- WHAT — specific, no vague "helps with"
- WHEN — triggers and contexts, including indirect signals
- INPUTS — types, constraints, sensible defaults
- RETURNS — concise vs detailed, error conditions with recovery
- MASKING — summarize raw tool output before passing forward

## Narrow fast tools beat god-tools

- **Anti-pattern:** 40+ tool definitions registered, eating half the context window. MCP god-tools with 2–5s round-trips. REST API wrappers turning every endpoint into a separate tool.
- **Cost:** "3x the tokens, 3x the latency, 3x the failure rate."

Concrete contrast:
- **Bad:** Chrome MCP doing screenshot → find → click → wait → read = 15 seconds end-to-end.
- **Good:** Playwright CLI doing each operation in 100ms = 75x faster.

Principle: **software doesn't have to be precious anymore.** Build the narrow tool you need (often a 50-line CLI). Don't reach for the kitchen-sink integration. The cost of a custom tool is now near zero; the cost of a slow/bloated tool is paid on every invocation forever.

## Tool design principles

Tools are **contracts**, not APIs. The agent cannot ask clarifying questions mid-call, so the description must carry the full usage story. Key rules:

- **Single clear purpose.** If a human engineer can't say definitively which tool to use, the agent can't either. Overlap → selection errors.
- **Consolidate over proliferate.** Fewer comprehensive tools outperform many specialized ones. Target 10–20 tools; use namespacing if you must go higher.
- **Descriptions are prompts.** They directly shape agent behavior. Every description must answer: *what it does*, *when to use it*, *what it returns*.
- **Verb-noun naming.** `search_invoices`, not `invoiceSearch` or `find`. Include a prefix when namespacing (`gmail_search_messages`).
- **Recovery-focused errors.** Error messages must state *what failed*, *why*, and *what to try next*. A stack trace isn't a signal; it's noise.
- **Dual response formats.** Offer concise (default) and detailed variants so the agent can pick based on how much context it's already carrying.
- **Iterate from observed failures.** Log which tool the agent reached for when it got the wrong result, and fix the description, not the agent.
