# Claude

The canonical version. This is the one used on real work, and the one the other three are adapted
from.

**File:** [`SKILL.md`](SKILL.md) · **Method:** [`../README.md`](../README.md)

## Install

```
~/.claude/skills/kql-grounded-queries/SKILL.md
```

The folder **must** be named `kql-grounded-queries` — it has to match the `name` in the skill's
frontmatter, or Claude will not load it. The project folder here is named for the agent, not the
skill, so this is a copy-and-rename rather than a straight copy:

```bash
mkdir -p ~/.claude/skills/kql-grounded-queries
cp SKILL.md ~/.claude/skills/kql-grounded-queries/SKILL.md
```

For a single project rather than everywhere, use `.claude/skills/` in the project root instead.

On claude.ai or Claude Desktop, paste the contents of `SKILL.md` into a Project's custom
instructions. The rules are identical; the triggering is manual rather than automatic.

## Triggering

Nothing to invoke. The `description` in the frontmatter fires on any KQL-shaped request, including
ones that never say "KQL" — *"find agents running on laptops in Defender"* is enough.

## Connect these too

| Server | What it adds |
|---|---|
| **Microsoft Learn MCP** | Grounded documentation search alongside direct page fetches. |
| **KQL Search MCP** (`https://www.kqlsearch.com/mcp`) | The community corpus for the prior-art step. |

Without them the skill still works — it falls back to fetching documentation URLs directly — but
the prior-art step gets thin.

## Checking it is working

Ask for a query against a table, and watch whether it fetches the schema page before answering. If
a query appears immediately with no lookup, the skill has not loaded. Check the folder name first.
