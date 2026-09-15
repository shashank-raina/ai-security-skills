# The KQL agent that refuses to guess

*What I built after one too many queries that looked right and were not.*

---

Ask a model for a KQL query and you get one straight away. Good formatting, sensible operators, and every so often a column that has never existed in that table.

I want to spend a bit of this post on why that happens, because "the AI made it up" is not really an explanation, and the actual reason changes what you can do about it.

## How a model invents a column name

A language model does not keep a copy of the Defender XDR schema anywhere. What it has is weights that encode which tokens tend to follow which other tokens. When it writes KQL, it is producing the most probable continuation of what came before. In something that starts `AgentsInfo | where`, tokens like `TimeGenerated`, `DeviceName` and `AccountName` are all very probable, because they show up constantly in the KQL it was trained on.

Probable and present are two different things. There is no separate store it consults. At no point during generation does it look up whether `AccountName` actually exists in that table. It emits the token because the token fits.

And nothing in the output tells you which case you are in. A column the model genuinely absorbed from documentation and a column it assembled because the parts looked right come out identical. Same confidence, same formatting. The model is not being careless or dishonest. It has no mechanism for telling the two apart, so neither do you.

Two things make this particularly bad for Microsoft security tables.

The training data argues with itself. Blog posts from 2024, GitHub repos nobody has touched in two years, retired Learn pages and current documentation all sit in the same pile. `AIAgentsInfo` appeared in a great deal of content before it was replaced by `AgentsInfo`. The retired name has more mass in the corpus, so it is more probable, so that is what comes out.

And these schemas move constantly. In the last year the agent inventory table was renamed, Learn and the deployed portal disagreed for months over whether it was `AgentInfo` or `AgentsInfo`, and four `EntraAgent*` identity tables shipped in preview with no published column reference at all. Every model has a cutoff sitting behind some of that, and a newer model just has a more recent snapshot of a moving target.

Asking it to check its own work does not help. "Are you sure that column exists?" produces a confident yes from exactly the same machinery that produced the column.

## Why this bites harder in KQL

When a made-up column breaks the query, you find out immediately. Advanced Hunting throws an error, you fix it, nothing is lost but a minute.

The one that hurts is quieter. The model picks a column that does exist and filters it on a value that never appears in it. The query runs fine. It returns nothing. Zero rows reads as a clean result — no risky agents, no suspicious activity — and it goes into a report.

I have shipped a bad query myself. Mine was a one-line reduction that collapsed a fleet of several hundred agents into a single row, and the workbook tile sat there reporting an inventory of one for longer than I want to admit. Nothing about it looked broken. That is what pushed me into building this.

## What I built

It is a Claude skill. A few hundred lines of instructions that sit in front of the model and stop it answering the way it wants to.

The rule everything else hangs off: never use a table or column name you have not verified in this session. Whatever the model thinks it knows about a schema is a starting guess and gets treated as one.

Before any query is written, it fetches the reference page for every table it intends to use. Log Analytics and Sentinel tables live at predictable Learn URLs, and so do the Defender XDR advanced hunting ones, so this is cheap. It pulls the exact column list off the page and works only from that.

If it cannot verify something, it stops and tells me. It does not reach for the closest plausible name. I would rather be told there is a gap than handed a query that runs and quietly means something else.

Everything comes back with sources. Which schema pages were checked, which existing queries were adapted, with links. I need to be able to show a customer where a detection came from.

It checks queries I paste in too. Plenty of KQL in circulation references columns that were renamed underneath it.

And it states its assumptions out loud — custom tables, connector coverage, licence-gated tables, ingestion lag.

One small thing I ended up liking: a 404 on a schema page is treated as information. If the reference URL does not resolve, either the table name is wrong or the table is custom, and both of those are useful answers.

## The half I did not build

There is one more rule: look for prior art before writing anything new. Detection engineering has years of public work behind it and an agent composing from first principles throws all of that away.

That rule runs on KQL Search, which pulls KQL queries published across GitHub into one searchable place. It has been a genuinely useful site for years.

What makes it work here is that Ugur Koc, who built and maintains it, also put an MCP server on top. My agent can search that corpus by table, keyword or technique, pull back a specific query and adapt it, without me stopping to go and browse anything.

The corpus was already good. The MCP server is what let an agent reach it. If you maintain something in this space and you have not put an MCP endpoint on it yet, it is worth the afternoon. Thank you, Ugur.

## What it costs

It is slow. Every table gets fetched, every column gets checked, and a query that would have appeared instantly takes a dozen tool calls. For poking around I do not bother with it.

For anything going into a workbook, a detection rule, a customer report or a post like this one, it is the only version I trust.

## Where it still falls over

Preview tables with no published schema are the hard case. Those four `EntraAgent*` tables have an announcement blog and nothing else. The agent cannot verify what has not been written down, so it flags the query as environment-dependent and wraps the columns in `column_ifexists()`. That is a hedge, and I know it.

Custom `*_CL` tables and workspace functions are the same shape. It asks me for the schema instead of assuming one, which is correct, but somebody still has to know the answer.

And verifying a table exists in the documentation says nothing about whether it is populated in your tenant. That is a licensing and connector question, and I have written about it separately.

---

There is no clever engineering in any of this. It is a set of instructions that refuse to let the model answer immediately, plus someone else's MCP server doing the part I could not have done alone.

*KQL Search is at [kqlsearch.com](https://www.kqlsearch.com/). Table names in this post reflect Microsoft Learn in mid-September 2026 and will age, which is the whole problem.*
