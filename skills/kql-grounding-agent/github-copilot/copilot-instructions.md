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
   - Index pages: `.../reference/tables-index` and `.../defender-xdr/advanced-hunting-schema-tables`
   Extract the exact column list. A URL that does not resolve means the table name is wrong or
   the table is custom — both are useful answers.
4. Search for existing published queries covering the same tables and goal, and adapt proven
   patterns rather than composing from nothing. [KQL Search](https://www.kqlsearch.com/) indexes
   KQL published across GitHub and is the fastest way to find them; the Azure-Sentinel repository
   is the other. Re-verify any columns those queries use — published queries go stale too.
   If your Copilot setup supports MCP servers, attaching KQL Search makes this a tool call rather
   than a web search.

### While writing

- Use only verified identifiers.
- Time filter first, high-selectivity `where` early, `has`/`has_any` in preference to `contains`,
  explicit join kinds, explicit final `project`.
- `TimeGenerated` in Log Analytics, `Timestamp` in Defender XDR. Tables streamed to both carry both.
- Handle dynamic columns per their documented type with `parse_json()`, `tostring()`, `mv-expand`.

### Before returning

Check every table is in the verified set, every column exists in that table's verified schema,
and every operator is supported on the target surface. Fix or report — do not ship past a failure.

### Never

- Substitute a plausible name for one you could not verify. Say so and stop.
- Present a query built on unverified identifiers as if it were grounded. If the user insists on
  one, mark it inline with `// UNVERIFIED` and list it in the assumptions.

### Always include

- The schema URLs you verified, and links to any queries you adapted.
- Assumptions: custom tables, connector coverage, licence-gated tables, ingestion lag.

### Custom tables

For `*_CL` tables and workspace functions, ask the user for the schema rather than assuming one,
and mark the result environment-dependent.
