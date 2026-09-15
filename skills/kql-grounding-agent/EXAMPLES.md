# Worked examples

Four failure modes this skill exists to catch, and what grounding does about each.

These illustrate the *class* of error rather than reproducing any particular model's output — the point is that all four produce queries that look completely reasonable.

---

## 1. Snapshot tables and the collapsed fleet

`AgentsInfo` stores repeated snapshots of every agent, so it has to be reduced to latest-state before anything is counted. The tempting reduction:

```kusto
AgentsInfo
| summarize arg_max(Timestamp, *)
```

That returns exactly one row — the most recently written row in the whole table. It is valid KQL, it runs, and it reports a fleet of one. I shipped this into a workbook tile and did not spot it for longer than I would like.

**Grounded version:**

```kusto
AgentsInfo
| summarize arg_max(Timestamp, *) by AgentId
| where LifecycleStatus != "Deleted"
| summarize Agents = count() by Platform
```

The `by AgentId` is the fix. The lifecycle filter matters too, or deleted agents stay in the count.

**What grounding contributes:** the column list comes off the schema page, so `LifecycleStatus` and its documented values (`Active`, `Blocked`, `Uninstalled`, `Deleted`) are verified rather than assumed. Microsoft's own published sample queries for the table use the `by AgentId` form, which the prior-art step surfaces before anything is composed.

Sources: [AgentsInfo schema](https://learn.microsoft.com/defender-xdr/advanced-hunting-agentsinfo-table) · [AgentsInfo sample queries](https://learn.microsoft.com/azure/azure-monitor/reference/queries/agentsinfo)

---

## 2. Retired table names outweigh their replacements

`AIAgentsInfo` was built for Copilot Studio scenarios and has been replaced by the unified `AgentsInfo` table. It remained accessible until 1 July 2026.

A model trained on content from before that transition has seen `AIAgentsInfo` far more often than `AgentsInfo`, which makes the retired name the more probable completion. The query it produces is syntactically fine and targets a table that is on its way out or already gone.

**What grounding contributes:** the table name is resolved against the advanced hunting schema index before use. The naming-changes page is itself documentation, so the transition is discoverable rather than something you find out when a scheduled rule stops returning rows.

Sources: [Advanced hunting schema — naming changes](https://learn.microsoft.com/defender-xdr/advanced-hunting-schema-changes) · [Schema tables index](https://learn.microsoft.com/defender-xdr/advanced-hunting-schema-tables)

---

## 3. The right query on the wrong surface

The same table can behave differently depending on where you query it, and some tables exist on only one surface.

- Advanced Hunting uses `Timestamp`. Log Analytics uses `TimeGenerated`. Tables streamed to both carry both.
- A table absent from the tenant schema returns a hard error on Advanced Hunting and on the data lake, rather than an empty result.
- `union isfuzzy=true`, the usual way of tolerating a missing table, is a Log Analytics behaviour.

A model that has learned KQL patterns generally will mix these, because in training data they all appear as normal KQL.

**What grounding contributes:** the surface is established during intent clarification, then every table is verified against that surface's documentation and the correct timestamp column is used. Where a request spans both, the skill produces two variants or restricts to the column intersection.

Sources: [Advanced hunting schema tables](https://learn.microsoft.com/defender-xdr/advanced-hunting-schema-tables) · [Azure Monitor table reference](https://learn.microsoft.com/azure/azure-monitor/reference/tables-index)

---

## 4. Preview tables with no published schema

The four `EntraAgent*` identity tables shipped in preview with no published column reference. There is nothing to verify against.

This is where the skill cannot deliver what it promises, and says so. It does not invent column names to fill the gap, and it does not pretend the result is grounded:

```kusto
EntraAgentIdentities
| extend BlueprintId = tostring(column_ifexists("agentIdentityBlueprintId", ""))
| summarize Agents = count() by BlueprintId
| top 10 by Agents
```

The `column_ifexists()` guard means a renamed column degrades to an empty string instead of breaking the query, and the whole thing is flagged as environment-dependent and requiring validation in-tenant. Read the output with that in mind: an empty string for every row is indistinguishable from a real empty value, so a filter on that column matches nothing or everything and a `summarize` on it collapses into one bucket. The query runs either way, which is what makes this a diagnostic rather than something to put in a workbook.

**What grounding contributes:** honesty about the boundary. A query marked "verify this against your own schema tab" is worth more than one presented with the same confidence as a verified one.

---

## What an answer looks like

Every response carries its provenance, so a query can be audited later or defended to a customer:

> **Query** — KQL targeting the confirmed surface
>
> **Sources verified** — each table with the schema URL that was fetched; each adapted community query with its link
>
> **Assumptions and environment dependencies** — custom tables, connector coverage, licence-gated tables, ingestion lag
>
> **Notes** — performance, tuning, portability between surfaces

If a table cannot be resolved, the answer says what was checked and asks whether it is a custom table, rather than returning something plausible.
