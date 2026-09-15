# KQL Grounding Agent

An agent that will not write a KQL query until it has verified every table and every column
against Microsoft's documentation.

Built for Microsoft Sentinel, Log Analytics and Defender XDR Advanced Hunting, where the schemas
move faster than any model's training data.

---

## Why a model invents a column name

Ask a model for a KQL query and you get one straight away. Good formatting, sensible operators,
and every so often a column that has never existed in that table.

A language model holds no copy of the Defender XDR schema. It has weights encoding which tokens
tend to follow which other tokens, so in something beginning `AgentsInfo | where`, tokens like
`TimeGenerated`, `DeviceName` and `AccountName` are all highly probable — they appear constantly
in the KQL it trained on. Probable and present are different things, and there is no step during
generation where the model checks which it is dealing with.

Nothing in the output tells you either. A column genuinely absorbed from documentation and one
assembled because the parts looked right come out identical: same confidence, same formatting.
Asking the model to double-check produces a confident yes from the same machinery that produced
the error.

Microsoft security tables make this sharper than most. Retired names carry more weight in the
training corpus than their replacements, so the old answer is often the more probable one. And
the schemas genuinely move — tables get renamed, documentation lags the portal, preview tables
ship with no published column reference at all.

The failure that costs you is the quiet one. A broken column name errors immediately and you fix
it. A real column filtered on a value that never appears runs fine, returns zero rows, and zero
rows reads as a clean result.

---

## How it works

One rule underneath everything:

> **Never emit a table or column name you have not verified in this session.**
> Training knowledge of a schema is a starting guess and gets treated as one.

Everything below exists to enforce that. The order matters — each step feeds the next, and none
of them can be skipped.

### 1. Establish the surface before anything else

Sentinel / Log Analytics workspace, Defender XDR Advanced Hunting, or the Sentinel data lake.
This decides which documentation is authoritative, which timestamp column is correct, and which
operators are available. If the request does not say, the agent states its assumption rather than
quietly picking one.

### 2. Name candidate tables, and treat them as unproven

The model lists what it thinks is relevant. At this point nothing has been established — this is
the hypothesis, written down so it can be checked.

### 3. Verify every table against its reference page

These URLs are deterministic, which is what makes the whole approach cheap enough to do every
time:

```
Log Analytics / Sentinel   https://learn.microsoft.com/azure/azure-monitor/reference/tables/{TableName}
Defender XDR               https://learn.microsoft.com/defender-xdr/advanced-hunting-{tablename}-table
Index pages                .../reference/tables-index
                           .../defender-xdr/advanced-hunting-schema-tables
```

Note the casing: Log Analytics uses the table's own casing, Defender XDR lowercases it. The exact
column list is read off the page and recorded, along with the URL.

**A 404 is information, not a failure.** If the reference page does not resolve, either the table
name is wrong or the table is custom. Both are answers, and both are more useful than a query.

### 4. Retrieve prior art before composing anything

Search the published corpus for queries against the verified tables, and adapt proven patterns
rather than writing from nothing. Detection engineering has years of public work behind it.

Columns in community queries get re-verified before use — published queries go stale in exactly
the same way training data does.

### 5. Compose using only verified identifiers

Time filter first. High-selectivity `where` early. `has` / `has_any` in preference to `contains`.
Explicit join kinds. An explicit final `project`. `TimeGenerated` on Log Analytics, `Timestamp`
on Defender XDR. Dynamic columns handled per their documented type with `parse_json()`,
`tostring()` and `mv-expand`.

### 6. Self-check line by line before returning

- Every table is in the verified set
- Every column exists in that table's verified schema, watching for Defender-versus-Sentinel
  differences on tables that exist in both
- Every operator is supported on the target surface
- The correct timestamp column is used throughout

Any failure gets fixed and re-checked, or reported as a gap. Nothing ships past a failed check.

### What comes back

Every answer carries its provenance, so a query can be audited later or defended to a customer:

> **Query** — KQL targeting the confirmed surface
>
> **Sources verified** — each table with the schema URL that was fetched; each adapted query with
> its link
>
> **Assumptions and environment dependencies** — custom tables, connector coverage, licence-gated
> tables, ingestion lag
>
> **Notes** — performance, tuning, portability between surfaces

### What it will not do

Substitute a plausible name for one it could not verify. If an identifier cannot be established,
the agent says what it checked and stops. Where a user insists on an unverified identifier, it
goes in marked `// UNVERIFIED` and listed in the assumptions, never silently.

It also verifies queries you paste in before modifying them. Plenty of KQL in circulation
references columns that were renamed underneath it.

---

## Getting it

Same rules, two delivery mechanisms.

| Platform | File | Status |
|---|---|---|
| **Claude** (Claude Code, claude.ai, Claude Desktop) | [`claude/SKILL.md`](claude/SKILL.md) | **In daily use.** Canonical version. |
| **GitHub Copilot** | [`github-copilot/copilot-instructions.md`](github-copilot/copilot-instructions.md) | Adaptation. Format follows Microsoft's documented custom-instructions feature; behaviour not yet validated to the same standard. |

**Claude:** the folder must match the skill's frontmatter name — copy `claude/SKILL.md` to
`kql-grounded-queries/SKILL.md` in your skills directory. It activates on any KQL request,
including ones that never use the word "KQL".

**GitHub Copilot:** place the file at `.github/copilot-instructions.md` in your repository, or at
`%USERPROFILE%/copilot-instructions.md` for user-level preferences that apply everywhere.

### Connections worth adding

| Connection | What it adds |
|---|---|
| [KQL Search MCP](https://www.kqlsearch.com/) | The community query corpus, searchable by table, keyword or technique. This is what makes step 4 work. |
| Microsoft Learn MCP | Grounded documentation search alongside direct page fetches. |

Without either, schema verification still works through direct documentation fetches; prior art
gets thinner.

---

## Worked examples

Four failure modes and what grounding does about each, in [EXAMPLES.md](EXAMPLES.md).

## Credit

Step 4 depends on [**KQL Search**](https://www.kqlsearch.com/), which aggregates KQL queries
published across GitHub into one searchable corpus.

It works here because **[Ugur Koc](https://ugurkoc.de/)**
([@ugurkocde](https://github.com/ugurkocde)), who built and maintains KQL Search, also put an MCP
server on top of it — so an agent can search that corpus mid-task instead of a human going off to
browse. The corpus was already good; the MCP server is what let an agent reach it.

## Known limits

- **Preview tables with no published schema.** Nothing can be verified against documentation that
  does not exist. These get flagged as environment-dependent, with columns guarded by
  `column_ifexists()`. That is a hedge, not grounding.
- **Custom `*_CL` tables and workspace functions.** The agent asks for the schema rather than
  assuming one, which means somebody still has to know it.
- **Documentation is not your tenant.** Verifying that a table exists says nothing about whether
  it is populated for you — that is a licensing and connector question. There is
  [an interactive grid](https://aiagentsecurity.guide/licensing.html) for that part.
