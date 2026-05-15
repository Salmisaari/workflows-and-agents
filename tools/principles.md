# Tools

Tools are the component of harness anatomy that turns the model's decisions into side effects. Without them the model is a text predictor; with them it's an agent.

Two categories cover almost everything.

**Built-in tools** are the runtime's core set — in Claude Code that's `Read`, `Edit`, `Glob`, `Grep`, `Write`, `Bash`. Fast, stable, narrow.

**MCP tools** come from Model Context Protocol servers — external catalogs that plug in at runtime, often exposing existing services (Gmail, Slack, Postgres).

Results from either kind flow back into context on the next turn, so tool output shape matters as much as tool behavior.

## Narrow fast tools beat god-tools

The pattern that shows up most in production: a small number of narrow, fast tools outperforms a large catalog of wide, slow ones.

The anti-pattern is 40+ tool definitions eating half the context window before the conversation starts, with MCP god-tools taking 2–5 seconds per call. Cost: "3× the tokens, 3× the latency, 3× the failure rate."

## Tools are contracts, not APIs

An API assumes the caller can ask clarifying questions, read docs, or retry with different arguments after an error.

Tools don't work that way. The agent calls the tool exactly once per turn, with whatever arguments it inferred from the description, and moves on.

Tool descriptions are not documentation; **they are prompts**. A vague description doesn't result in "the agent asks for clarification" — it results in the wrong tool being invoked, or the right tool being invoked wrongly.

Tools are built with five dimensions to make a description carry its weight:

- **WHAT** — specific action. "Searches Gmail messages matching a query" is a contract; "works with email" is a hint the agent has to guess at.
- **WHEN** — triggers and indirect signals ("user mentions an email thread, even without saying 'Gmail'").
- **INPUTS** — types, constraints, sensible defaults.
- **RETURNS** — output shape, concise vs. detailed variants, and what failure looks like with how to recover.
- **MASKING** — summarize raw output before passing it forward. A 10KB log dump in context poisons the next turn; a three-line summary keeps attention sharp.

## Tools live above the cache breakpoint

The tool surface — names, descriptions, and input schemas exposed this session — sits above the prompt-cache breakpoint, in the same stable prefix as the system prompt (see `../harness/context-injection/principles.md`). After the first turn the cache reads this prefix at a fraction of full input cost and latency. The implication is that changing the tool surface is not a local quality change; it is a cache-invalidation event for every subsequent call in every active session.

Dynamic tool loading — varying which tools are exposed turn by turn based on intent classification — sounds elegant and is usually the wrong call in production. Every tool-set change forces the rest of the thread to pay full input cost, which dwarfs the token savings from showing fewer tools. The pattern that holds up is the opposite: define a stable surface up front, accept that some tools will be unused in some sessions, and pay the small attention cost rather than the large cache cost.

Two operational consequences follow. **Tool surface changes are blast-radius events** — renaming a tool, reordering parameters, or rewriting a description ripples through every cached conversation and every downstream eval, so treat changes like a schema migration, not a docstring edit. And **regression-test tool changes before shipping** — the model's tool selection on path A can shift when you "improve" a tool on path B, so run the eval set that exercises adjacent tools whenever you touch the surface. "I just fixed the description" hides the most expensive regressions.

The cache principle pairs with the design principles below: "Single clear purpose" keeps overlap out of the surface; cache stability keeps the surface stable once it's right.

## Design principles

Rules that survive in production:

- **Single clear purpose.** If an engineer can't say definitively which of two tools to use, the agent can't either. Overlap → selection errors. The fix is usually to merge or delete.
- **Consolidate over proliferate.** Target maximum of 10–20 tools. If you need more, namespace (`gmail_search_messages`, `slack_send_message`) so the agent can filter by prefix.
- **Verb-noun naming.** `search_invoices` tells the agent what this does. `invoiceSearch` is camelCase noise. Single-verb names (`find`, `get`) don't differentiate from siblings.
- **Recovery-focused errors.** `ConnectionRefusedError: [Errno 111]` is noise. "Failed to reach the Gmail API: token expired. Try refreshing." tells the agent what to do next. Errors are part of the contract.
- **Dual response formats.** Concise default, detailed variant on request. Cheap to offer, pays back repeatedly.
- **Iterate from observed failures.** When the agent reaches for the wrong tool, the fix is almost always the description, not the agent.

**The anti-pattern:** hard-coding tool behavior that should be in the description. Subtle rules ("never call this in a preview environment") belong in the description, not in a wrapper. The agent reads the description; wrapper logic is invisible to it.
