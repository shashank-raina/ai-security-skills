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
| Log Analytics / Sentinel table schemas | `web_fetch` the deterministic URL: `https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/{tablename}` (lowercase, e.g. `signinlogs`). Table discovery index: `https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables-index`. The **Microsoft Learn MCP server**, if connected, may also be used for grounded doc search. |
| Defender XDR Advanced Hunting schemas | `web_fetch`: `https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-{tablename}-table` (lowercase). Table list: `https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-schema-tables`. |
| Community query corpus | **KQL Search MCP** (`kqlsearch.com/mcp`) if connected — search by table, keyword, technique. If not connected, tell the user they can add it as a custom connector, and fall back to `web_search` scoped conceptually to kqlsearch.com and community blogs. |
| Official Microsoft queries | Azure-Sentinel GitHub repo. Use `web_search` (e.g. `Azure-Sentinel github EmailUrlInfo hunting query`) then `web_fetch` the raw file. Key paths: `Detections/`, `Hunting Queries/`, `Solutions/{Name}/Analytic Rules/`. Rule YAML declares `requiredDataConnectors`, tactics, techniques. |
| KQL language reference | `web_fetch` `https://learn.microsoft.com/en-us/kusto/query/{operator-name}` if unsure an operator/function exists or is supported in Log Analytics (some Kusto features are ADX-only). |

A failed lookup and a documented absence are different things. A timeout, a 403, a moved URL template — and a 404, which Learn serves for a slug it does not recognise as readily as for a table that does not exist — say nothing on their own about the schema: retry once, go to the index page, and if you cannot read one either, report the lookup as failed rather than drawing a conclusion. Absence is established by a page you actually read — an index that lists the surface's schema and does not list your table. That is *signal*, not failure. It means one of three things: the name is wrong, the table is custom to the tenant, or the table is in preview with no reference published yet. Say which the evidence supports; where nothing distinguishes them, report the schema as unavailable rather than picking one.

## Mandatory workflow (in order, no skipping)

1. **Clarify intent.** Goal, query surface (Sentinel workspace vs Defender XDR vs both), time window, stated constraints. Ask only if genuinely ambiguous; otherwise state your interpretation as an assumption.
2. **Resolve candidate tables.** List tables you believe are relevant; note which surface each belongs to. Still hypothetical.
3. **Verify schemas.** If no fetch, browse or documentation tool is available in this session, stop here: say verification is not possible, hand over the reference URLs so the user can check the schema themselves, and do not write the query. What you remember about these schemas is not a substitute for the lookup, and a query from recall presented as grounded is the failure this skill exists to prevent. Otherwise, fetch each table's reference page. Extract the exact columns you intend to use; record the URL. On 404: check the index pages for the correct name. If the name is in the index but the page is missing, treat the schema as unpublished, not wrong. If it is nowhere, ask whether it's a custom `*_CL` table and request its schema from the user. Custom tables, workspace functions and unpublished preview tables proceed only on user-supplied schema, marked environment-dependent.
4. **Retrieve prior art, where you can.** Query KQL Search MCP and the Azure-Sentinel repo for the verified tables + goal keywords. Search on table names and generic technique keywords only — this step leaves the tenant. Account names, UPNs, hostnames, IP addresses, internal domains, ticket references and anything pasted from the user's own data stay out of the search string. Custom identifiers are not safe to send either: a `*_CL` table or a workspace function can carry a project, a client or a case in the name, so never look one up externally — verify it from the schema tab or a `getschema` run, with a redacted name where even that is sensitive. Re-verify any columns the found queries use before adapting — community queries can be stale. This step depends on a search capability you may not have: if none is available, say the prior-art step was skipped and compose from the verified schemas alone. Never cite an adapted query you did not actually retrieve.
5. **Compose.** Use only verified identifiers. Style: time filter first, high-selectivity `where` early, `has`/`has_any` over `contains` for whole-term matches but never where substring semantics are needed — inside a URL, path, command line or domain fragment `contains` is the correct operator, and say which you chose — explicit join kinds, explicit final `project`, `//` comments on non-obvious logic, entity-mappable columns surfaced for detections.
6. **Self-check** the final query line by line before returning:
   - Every table is in the verified set
   - Every **source** column exists in its table's verified schema (watch XDR-vs-Sentinel column differences for dual-surface tables)
   - Every name the query itself creates is defined before use in the query's own dataflow and not shadowed later — `extend` results, `summarize` outputs, a renaming `project`, and the columns a join invents when both sides carry the same name, where the right-hand one returns with a numeric suffix (`Key` and `Key1`). `$left` and `$right` belong to the `on` clause and cannot be projected. These need no external verification; do not report them as gaps
   - Every operator/function is supported on the target surface
   - Dynamic columns handled per documented type (`parse_json()`, `tostring()`, `mv-expand`)
   - Correct timestamp column for the surface, never carried across: `TimeGenerated` (Log Analytics) vs `Timestamp` (Defender XDR); the Sentinel data lake reads its own — `TimeGenerated` is usual there but federated tables may lack it, and asset tables also carry `_SnapshotTime` and `_ReceivedTime`, so read it off the schema like any other column

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
- **Verifying a pasted query** means reading its identifiers, not searching for its contents. An incident query can carry the incident in it.
- **User insists on an unverified identifier:** a bare assertion is a claim about their environment, not a source, and it does not put the name in the query. Ask for the schema — the portal's schema tab or a `getschema` run — which makes it verifiable against their tenant and marks the result environment-dependent. Until then the answer is the gap. Never put an unverified identifier in on your own initiative; an annotated guess is still a guess, and the query outlives the annotation once it is copied.
- **Cross-surface request:** verify on both surfaces; produce two variants or restrict to the column intersection.
