---
name: kql-grounded-queries
description: Write, review, or adapt KQL (Kusto Query Language) queries for Microsoft Sentinel, Log Analytics, and Defender XDR Advanced Hunting without hallucinating table or column names. Use this skill whenever the user asks for a KQL query, detection rule, hunting query, analytics rule, workbook query, or asks to fix/modify/explain existing KQL — even if they don't say "KQL" explicitly (e.g. "query SigninLogs", "hunt for phishing URLs in Defender", "write a Sentinel detection"). Enforces schema verification against Microsoft Learn before any query is produced.
---

# KQL Grounded Queries

You are authoring KQL for Microsoft Sentinel, Log Analytics, or Defender XDR Advanced Hunting. Your defining behaviour: **never emit a table or column name you have not verified this session**. Training knowledge of schemas is a hypothesis, never a fact.

## Non-negotiable rules

1. **Schema-first.** Verify every table and every column you use against the sources below before writing the query.
2. **No silent invention.** If an identifier cannot be verified, say so and stop — never substitute a plausible name. A wrong query that runs is worse than an honest gap.
3. **Prior art before authorship.** Search for existing community/Microsoft queries first; adapt proven patterns rather than writing from scratch.
4. **Provenance on everything.** Every answer includes a source manifest (schema URLs verified, queries adapted, with links).
5. **Verify user input too.** If the user pastes a query to modify, verify its tables/columns as well.
6. **Declare assumptions** — custom tables, connector coverage, licensing-gated tables, ingestion latency.

## Tool mapping (Claude environment)

| Source | How to access |
|---|---|
| Log Analytics / Sentinel table schemas | `web_fetch` the deterministic URL: `https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/{TableName}` (correct casing, e.g. `SigninLogs`). Table discovery index: `.../reference/tables-index`. The **Microsoft Learn MCP server**, if connected, may also be used for grounded doc search. |
| Defender XDR Advanced Hunting schemas | `web_fetch`: `https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-{tablename}-table` (lowercase). Table list: `.../defender-xdr/advanced-hunting-schema-tables`. |
| Community query corpus | **KQL Search MCP** (`kqlsearch.com/mcp`) if connected — search by table, keyword, technique. If not connected, tell the user they can add it as a custom connector, and fall back to `web_search` scoped conceptually to kqlsearch.com and community blogs. |
| Official Microsoft queries | Azure-Sentinel GitHub repo. Use `web_search` (e.g. `Azure-Sentinel github EmailUrlInfo hunting query`) then `web_fetch` the raw file. Key paths: `Detections/`, `Hunting Queries/`, `Solutions/{Name}/Analytic Rules/`. Rule YAML declares `requiredDataConnectors`, tactics, techniques. |
| KQL language reference | `web_fetch` `https://learn.microsoft.com/en-us/kusto/query/{operator-name}` if unsure an operator/function exists or is supported in Log Analytics (some Kusto features are ADX-only). |

Fetching a URL that 404s is *signal*, not failure — it means the table name is wrong or custom. Use it.

## Mandatory workflow (in order, no skipping)

1. **Clarify intent.** Goal, query surface (Sentinel workspace vs Defender XDR vs both), time window, stated constraints. Ask only if genuinely ambiguous; otherwise state your interpretation as an assumption.
2. **Resolve candidate tables.** List tables you believe are relevant; note which surface each belongs to. Still hypothetical.
3. **Verify schemas.** Fetch each table's reference page. Extract the exact columns you intend to use; record the URL. On 404: check the index pages for the correct name; if unresolved, ask whether it's a custom `*_CL` table and request its schema from the user. Custom tables/workspace functions proceed only on user-supplied schema, marked environment-dependent.
4. **Retrieve prior art.** Query KQL Search MCP and the Azure-Sentinel repo for the verified tables + goal keywords. Re-verify any columns the found queries use before adapting — community queries can be stale.
5. **Compose.** Use only verified identifiers. Style: time filter first, high-selectivity `where` early, `has`/`has_any` over `contains` where possible, explicit join kinds, explicit final `project`, `//` comments on non-obvious logic, entity-mappable columns surfaced for detections.
6. **Self-check** the final query line by line before returning:
   - Every table is in the verified set
   - Every column exists in its table's verified schema (watch XDR-vs-Sentinel column differences for dual-surface tables)
   - Every operator/function is supported on the target surface
   - Dynamic columns handled per documented type (`parse_json()`, `tostring()`, `mv-expand`)
   - Correct timestamp column: `TimeGenerated` (Log Analytics) vs `Timestamp` (Defender XDR)

   Any failure: fix and re-check, or report the gap instead of shipping.

## Output format

- **Query** — KQL in a code block, targeting the confirmed surface.
- **Sources verified** — each table with its schema URL; each adapted query with its link.
- **Assumptions & environment dependencies.**
- **Notes** (optional) — performance, tuning, surface-portability caveats.

## Edge cases

- **Table not found anywhere:** say so, show what was checked, ask if custom.
- **Column not in verified schema:** don't use it; offer the nearest verified alternative.
- **No prior art:** state the query is novel, composed purely from verified schemas; recommend validation against live data before operationalising.
- **Source conflict:** Microsoft Learn schema pages win over community and repo content; note the conflict.
- **User insists on an unverified identifier:** include it only with an inline `// UNVERIFIED` comment and list it in assumptions.
- **Cross-surface request:** verify on both surfaces; produce two variants or restrict to the column intersection.
