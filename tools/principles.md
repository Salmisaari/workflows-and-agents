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

## Design principles

Rules that survive in production:

- **Single clear purpose.** If an engineer can't say definitively which of two tools to use, the agent can't either. Overlap → selection errors. The fix is usually to merge or delete.
- **Consolidate over proliferate.** Target maximum of 10–20 tools. If you need more, namespace (`gmail_search_messages`, `slack_send_message`) so the agent can filter by prefix.
- **Verb-noun naming.** `search_invoices` tells the agent what this does. `invoiceSearch` is camelCase noise. Single-verb names (`find`, `get`) don't differentiate from siblings.
- **Recovery-focused errors.** `ConnectionRefusedError: [Errno 111]` is noise. "Failed to reach the Gmail API: token expired. Try refreshing." tells the agent what to do next. Errors are part of the contract.
- **Dual response formats.** Concise default, detailed variant on request. Cheap to offer, pays back repeatedly.
- **Iterate from observed failures.** When the agent reaches for the wrong tool, the fix is almost always the description, not the agent.

**The anti-pattern:** hard-coding tool behavior that should be in the description. Subtle rules ("never call this in a preview environment") belong in the description, not in a wrapper. The agent reads the description; wrapper logic is invisible to it.
