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
Log Analytics / Sentinel   https://learn.microsoft.com/azure/azure-monitor/reference/tables/{tablename}
Defender XDR               https://learn.microsoft.com/defender-xdr/advanced-hunting-{tablename}-table
Index pages                https://learn.microsoft.com/azure/azure-monitor/reference/tables-index
                           https://learn.microsoft.com/defender-xdr/advanced-hunting-schema-tables
```

Both paths are lowercase. The exact
column list is read off the page and recorded, along with the URL.

**A 404 is not a verdict.** Learn serves one for a slug it does not recognise as readily as for a
table that does not exist, so a 404 on its own — like a timeout or a 403 — means the lookup
failed, and the agent says so. What settles it is the index page: a schema index that does not
list your table is evidence, and then the name is wrong, the table is custom, or it is in preview
with nothing published yet. Any of those three is more useful than a query.

### 4. Retrieve prior art before composing anything

Search the published corpus for queries against the verified tables, and adapt proven patterns
rather than writing from nothing. Detection engineering has years of public work behind it.

Columns in community queries get re-verified before use — published queries go stale in exactly
the same way training data does.

### 5. Compose using only verified identifiers

Time filter first. High-selectivity `where` early. Explicit join kinds. An explicit final
`project`. Dynamic columns handled per their documented type with `parse_json()`, `tostring()`
and `mv-expand`.

`has` and `has_any` index whole terms and are faster, but they are not a drop-in replacement for
`contains` — anything matching inside a URL, a path or a command line needs substring semantics,
and the agent says which it used.

The timestamp column belongs to the surface and is never carried across: `TimeGenerated` on Log
Analytics, `Timestamp` on Defender XDR, and on the data lake whatever the table documents —
`TimeGenerated` is usual there but federated tables may lack it, and asset tables also carry
`_SnapshotTime` and `_ReceivedTime`.

### 6. Self-check line by line before returning

- Every table is in the verified set
- Every **source** column exists in that table's verified schema, watching for
  Defender-versus-Sentinel differences on tables that exist in both
- Every name the query creates for itself is defined before it is used and not shadowed later —
  `extend` results, `summarize` outputs, a renaming `project`, and the suffixed duplicate a join
  returns when both sides share a column name (`Key` and `Key1`). Those are dataflow checks, not
  schema gaps
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

Substitute a plausible name for one it could not verify, or put an unverified identifier into a
query — its own or yours. If an identifier cannot be established, the agent says what it checked
and stops. Telling it a column exists is a claim about your environment rather than a source, so
it asks for the schema tab or a `getschema` run instead; supply one and the identifier is verified
against your tenant, with the result marked environment-dependent throughout.

It also verifies queries you paste in before modifying them. Plenty of KQL in circulation
references columns that were renamed underneath it.

---

## Getting it

Same method, four delivery mechanisms.

| Platform | Where it goes | How it verifies |
|---|---|---|
| **Claude** (Claude Code, claude.ai, Desktop) | [`claude/SKILL.md`](claude/SKILL.md) | Fetches the schema page. **In daily use — canonical version.** |
| **Copilot Studio** | [`copilot-studio/`](copilot-studio/) | Agent instructions plus MCP tools, so lookups are real tool calls. Steps from current documentation; not yet built end to end. |
| **GitHub Copilot** | [`github-copilot/copilot-instructions.md`](github-copilot/copilot-instructions.md) | Fetches documentation URLs. Format per Microsoft's custom-instructions feature; not yet validated. |
| **Microsoft 365 Copilot** | [`m365-copilot/`](m365-copilot/) | Declarative agent using scoped web search, which searches rather than fetches. Read the limitation note before relying on it. |

The difference that matters between them is step 3. Claude, Copilot Studio and GitHub Copilot can
retrieve a specific documentation page and the schema index behind it, which is what lets a
missing table be established rather than guessed at.
A declarative agent relying on scoped web search only gets what an index returns, which is a
weaker promise — worth understanding before telling anyone the output is grounded. Attaching a
Microsoft Learn MCP plugin closes most of that gap.

**Claude:** the folder must match the skill's frontmatter name — copy `claude/SKILL.md` to
`kql-grounded-queries/SKILL.md` in your skills directory. It activates on any KQL request,
including ones that never use the word "KQL".

**GitHub Copilot:** place the file at `.github/copilot-instructions.md` in your repository, or at
`%USERPROFILE%/copilot-instructions.md` for user-level preferences that apply everywhere.

**Copilot Studio and Microsoft 365 Copilot** are agent-building platforms rather than instruction
files. Each folder carries the setup steps and the instruction text to paste in — including the
gotchas that stop the verification working, such as generative orchestration needing to be on
before Copilot Studio can use MCP at all.

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
published across GitHub into one searchable corpus. All four versions use it: as an MCP server on
Claude and Copilot Studio, as a plugin or scoped search site on Microsoft 365 Copilot, and as a
search target on GitHub Copilot.

It works here because **[Ugur Koc](https://ugurkoc.de/)**
([@ugurkocde](https://github.com/ugurkocde)), who built and maintains KQL Search, also put an MCP
server on top of it — so an agent can search that corpus mid-task instead of a human going off to
browse. The corpus was already good; the MCP server is what let an agent reach it.

## Known limits

- **Preview tables with no published schema.** Nothing can be verified against documentation that
  does not exist. The agent asks you for the schema — the portal's schema tab, or a `getschema`
  run — and stops if you do not have it. Ask and it will produce a `column_ifexists()` version,
  but that is a diagnostic rather than a query to keep: a column that is not there resolves
  silently to the default for every row, so a filter on it matches nothing or everything and a
  `summarize` collapses into one bucket. It runs, and what comes back is a false negative wearing
  the shape of a real result.
- **Hosts that cannot look anything up.** The whole approach rests on reading a documentation
  page. Where the agent has no fetch, browse or documentation tool at all, it says verification is
  not possible and stops — which is the right answer, but it does mean the skill is only as good
  as the tools around it.
- **Custom `*_CL` tables and workspace functions.** The agent asks for the schema rather than
  assuming one, which means somebody still has to know it.
- **Documentation is not your tenant.** Verifying that a table exists says nothing about whether
  it is populated for you — that is a licensing and connector question. There is
  [an interactive grid](https://aiagentsecurity.guide/licensing.html) for that part.
