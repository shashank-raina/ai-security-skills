# Microsoft 365 Copilot

A declarative agent, built either in **Agent Builder** inside Microsoft 365 Copilot or with the
Microsoft 365 Agents Toolkit. The grounding rules go in the agent's instructions; the verification
sources come from capabilities and plugins.

> Read the honest limitation below before choosing this route.

## The limitation that matters here

The Claude and Copilot Studio versions **fetch a documentation page**. A declarative agent's web
search capability **searches an index**, scoped to sites you nominate. Those are not the same
guarantee. Search returns what looks relevant; a fetch returns the schema page itself, or lets you
read the index that settles whether the table exists at all.

You can close most of that gap by attaching a plugin built on the Microsoft Learn MCP server,
which turns lookups back into real tool calls. Without one, treat this version as a strong prompt
rather than genuine grounding, and keep the habit of checking the schema tab yourself.

## 1. Create the agent

In Microsoft 365 Copilot, **Create an agent** (Agent Builder), or scaffold one with the
[Agents Toolkit](https://aka.ms/M365AgentsToolkit). Paste the instructions block below.

## 2. Scope web search to the documentation

Add the **web search** capability and restrict it with the `sites` array. **The array takes at
most four sites**, so spend them deliberately:

```json
{
  "capabilities": [
    {
      "name": "WebSearch",
      "sites": [
        { "url": "https://learn.microsoft.com/azure/azure-monitor/reference/tables" },
        { "url": "https://learn.microsoft.com/defender-xdr" },
        { "url": "https://www.kqlsearch.com" }
      ]
    }
  ]
}
```

Omitting `sites` lets the agent search anything, which reintroduces exactly the stale blog content
the grounding is meant to avoid. Scope it.

In Agent Builder the equivalent is the **Web content** toggle in the Knowledge pane. Note that the
tenant policy **Allow web search in Copilot** overrides that toggle — if an admin has disabled web
content, the toggle still appears enabled but the agent gets nothing.

## 3. Add a plugin for real lookups

Plugins let a declarative agent call an **MCP server** or a REST API, which is how you get from
searching to verifying. Build one from the Microsoft Learn MCP server, and a second from KQL
Search for the prior-art step. The Agents Toolkit generates plugin packages directly from an MCP
server or OpenAPI description.

Keep the count low. Up to five plugins are always injected into the prompt; past five the agent
falls back to semantic matching on the plugin description. Response quality degrades beyond about
ten functions or tools in total, and large tool output is truncated by the token window — which
matters when a schema page is long.

## 4. Test it

Ask for a query against a table you know was recently renamed. If the agent answers without
consulting anything, the grounding is not working — check that web search is scoped, the plugin is
attached, and the tenant policy is not blocking web content.

---

## Agent instructions

Declarative agent instructions are length-capped, so this is the condensed version. If you have
room, the fuller wording in [`../copilot-studio/README.md`](../copilot-studio/README.md) is better.

```text
You write KQL for Microsoft Sentinel, Log Analytics and Defender XDR Advanced Hunting.

Never give the user a table or column name you have not checked against Microsoft Learn
documentation in this conversation. What you think you know about these schemas is a guess —
they change often, and retired table names appear more in training data than their replacements.

Every time:

1. Confirm the target surface (Sentinel / Log Analytics, Defender XDR Advanced Hunting, or the
   Sentinel data lake). State your assumption if the user did not say.
2. Look up each table you intend to use in the Microsoft Learn documentation and read off its
   exact columns before writing anything.
3. Look for published queries against those tables on KQL Search (kqlsearch.com) and adapt
   them, re-checking their columns because published queries go stale too.
4. Write the query using only identifiers you checked. TimeGenerated for Log Analytics,
   Timestamp for Defender XDR, and on the Sentinel data lake whatever the table's own schema
   documents. Time filter first, selective filters early. Use has and has_any for whole-term
   matches and contains where the match is inside a URL, a path or a command line.
5. Re-check every table and column in your answer before sending it.

End every answer with what you verified and where, plus your assumptions: custom tables,
connector coverage, licensing, ingestion lag.

If you cannot verify an identifier, say so and stop. Never substitute one that looks right. Tell
the user plainly when you are unsure, and recommend they confirm against the schema tab in their
own portal.
```
