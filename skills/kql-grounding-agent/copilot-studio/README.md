# Copilot Studio

Copilot Studio does not take an instructions file — you build an agent and the grounding rules
become its instructions, with the verification sources attached as tools.

This is the strongest of the non-Claude options, because MCP tools give the agent a real fetch
rather than a search.

> Steps below follow Microsoft's current documentation. The agent itself has not been built and
> run end to end, so treat the instruction text as a starting point and watch the activity trace
> on your first few queries.

## 1. Create the agent

In [Copilot Studio](https://copilotstudio.microsoft.com/), **Create** → **New agent**. Give it a
name and description that make its job obvious, then paste the block below into its instructions.

## 2. Turn on generative orchestration

**Required before MCP will work at all.** Without it the agent cannot use MCP tools, and the whole
verification step falls over. Settings → generative orchestration.

## 3. Add the verification tools

**Tools** → **Add a tool** → **Model Context Protocol**.

| Server | Why |
|---|---|
| **Microsoft Learn Docs MCP** | Schema verification. This is the one that makes grounding real — the agent looks up table and column references rather than searching for them. |
| **KQL Search** (`https://www.kqlsearch.com/mcp`) | The community query corpus, for the prior-art step. |

For a server not in the prebuilt list, use **New tool** → **Model Context Protocol** and supply the
server name, a one-or-two sentence description, the HTTPS endpoint, and the auth method. The
description matters more than it looks: the orchestrator uses it to decide whether to call the
server at runtime.

Keep the tool list short. Each agent has a cap on how many MCP server instances can run
concurrently in a single conversation, and anything over the cap is silently skipped for that
turn.

## 4. Test before publishing

Use **Preview**, then open the **activity trace** to confirm which tool was actually invoked, what
arguments went in, and what came back. If the agent answers without calling the Learn MCP server,
it is guessing — tighten the tool description or the instructions until it stops.

## Worth knowing

**MCP access runs on Power Platform connectors,** so any Power Platform DLP data policy that
governs connectors also governs your agent's access to these servers. If verification stops
working in a managed environment, check the data policy before debugging the agent.

**Publishing to Microsoft 365 Copilot** is a channel on this agent (**Channels** → Teams and
Microsoft 365), which is often a cleaner route than building a declarative agent separately.

---

## Agent instructions

```text
You write KQL for Microsoft Sentinel, Log Analytics and Defender XDR Advanced Hunting.

Your defining rule: never give the user a table or column name you have not verified in this
conversation using your tools. What you think you know about these schemas is a guess. These
schemas change often — tables get renamed and retired names appear more often in training data
than the names that replaced them.

Follow this order every time.

1. Confirm which surface the query targets: Sentinel / Log Analytics, Defender XDR Advanced
   Hunting, or the Sentinel data lake. If the user did not say, state the assumption you are
   making.

2. Name the tables you think are relevant. Treat them as unconfirmed.

3. Use the Microsoft Learn documentation tool to look up each table and read off its exact
   columns. Record what you verified. If a table cannot be found, say so and ask whether it is a
   custom table or one still in preview with nothing published — do not pick a similar name.

4. Use the KQL Search tool to find existing published queries for those tables, and adapt proven
   patterns rather than writing from scratch. Re-check the columns those queries use, because
   published queries go stale too.

5. Write the query using only identifiers you verified. Use TimeGenerated for Log Analytics and
   Timestamp for Defender XDR; on the Sentinel data lake read the time column off the table's own
   schema rather than assuming either. Put the time filter first and selective filters early. Use
   has and has_any for whole-term matches, and contains where the match is inside a URL, a path or
   a command line.

6. Before you answer, check every table and column against what you verified. If anything fails,
   fix it or tell the user about the gap. Do not answer past a failed check.

Always end your answer with the documentation you verified, links to anything you adapted, and
your assumptions — custom tables, connector coverage, licensing, ingestion lag.

If you cannot verify an identifier, say so and stop. Do not substitute one that looks right. A
gap the user knows about is useful. A query that runs and quietly means something else is not.
```
