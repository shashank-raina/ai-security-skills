# KQL grounding instructions

> Adapted from the Claude skill in `../claude/SKILL.md`, which is the canonical version.
> The method it implements is described in `../README.md`.
> **Not yet tested in anger.** The file format is per Microsoft's documented custom-instructions
> feature; the behaviour it produces has not been validated the way the Claude version has.
> Treat it as a starting point and check the output.

## Writing KQL

When writing, fixing or explaining KQL for Microsoft Sentinel, Log Analytics or Defender XDR
Advanced Hunting, never use a table or column name you have not verified in this session.
Anything you believe about a schema from training is a starting guess.

### Before writing a query

1. Establish which surface the query targets: Sentinel / Log Analytics workspace, Defender XDR
   Advanced Hunting, or the Sentinel data lake. State the assumption if it is not given.
2. List the tables you believe are relevant. These remain unverified.
3. Verify each one against Microsoft documentation:
   - Log Analytics and Sentinel: `https://learn.microsoft.com/azure/azure-monitor/reference/tables/{TableName}`
   - Defender XDR Advanced Hunting: `https://learn.microsoft.com/defender-xdr/advanced-hunting-{tablename}-table` (lowercase)
   - Index pages: `https://learn.microsoft.com/azure/azure-monitor/reference/tables-index` and
     `https://learn.microsoft.com/defender-xdr/advanced-hunting-schema-tables`
   Extract the exact column list. A fetch that fails — timeout, 403, moved template — says nothing
   about the schema: retry once, try the index, then report the lookup as failed. A page that comes
   back without the table means the name is wrong, the table is custom, or it is in preview with
   nothing published yet. Say which the evidence supports.
4. Search for existing published queries covering the same tables and goal, and adapt proven
   patterns rather than composing from nothing. [KQL Search](https://www.kqlsearch.com/) indexes
   KQL published across GitHub and is the fastest way to find them; the Azure-Sentinel repository
   is the other. Re-verify any columns those queries use — published queries go stale too.
   If your Copilot setup supports MCP servers, attaching KQL Search makes this a tool call rather
   than a web search.

### While writing

- Use only verified identifiers.
- Time filter first, high-selectivity `where` early, explicit join kinds, explicit final `project`.
- `has`/`has_any` for whole-term matches, `contains` where substring semantics are needed — inside a
  URL, path, command line or domain fragment. Say which you chose and why.
- `TimeGenerated` in Log Analytics, `Timestamp` in Defender XDR. Tables streamed to both carry both.
  The Sentinel data lake reads its own: `TimeGenerated` is usual but federated tables may lack it,
  and asset tables also carry `_SnapshotTime` and `_ReceivedTime`. Never carry a convention across.
- Handle dynamic columns per their documented type with `parse_json()`, `tostring()`, `mv-expand`.

### Before returning

Check every table is in the verified set, every **source** column exists in that table's verified
schema, and every operator is supported on the target surface. Names the query itself creates with
`extend`, `summarize` or a renaming `project` are checked against the query's own dataflow instead —
defined before use, not shadowed — and are not schema gaps. Fix or report — do not ship past a failure.

### Never

- Substitute a plausible name for one you could not verify. Say so and stop.
- Put an unverified identifier into a query — yours or the user's. An annotated guess is still a
  guess, and the query outlives the annotation once it is copied. A user's bare assertion is a
  claim about their environment, not a source: ask for the schema tab or a `getschema` run, which
  makes it verifiable against their tenant, and report the gap until they supply one.

### Always include

- The schema URLs you verified, and links to any queries you adapted.
- Assumptions: custom tables, connector coverage, licence-gated tables, ingestion lag.

### Custom tables

For `*_CL` tables and workspace functions, ask the user for the schema rather than assuming one,
and mark the result environment-dependent.
