# AI Security Skills

Agent skills for working on Microsoft AI and agent security, built around one idea: an agent should verify what it claims rather than producing something that looks right.

Everything here is used on real work. The skills are plain Markdown instruction files — no framework, no dependencies.

---

## Skills

| Skill | What it does | Platforms |
|---|---|---|
| [**kql-grounding-agent**](skills/kql-grounding-agent) | Writes KQL for Sentinel, Log Analytics and Defender XDR without inventing table or column names. Checks every identifier against its real source — Microsoft documentation, or your own tenant's schema where the table is custom — and stops instead of guessing when it cannot. | Claude · Copilot Studio · GitHub Copilot · M365 Copilot |

More will land here as they prove themselves in use.

## Why grounding

A language model holds no copy of the Defender XDR schema. It has weights encoding which tokens follow which other tokens, so in a query beginning `AgentsInfo | where`, names like `TimeGenerated`, `DeviceName` and `AccountName` are all highly probable because they appear constantly in the KQL it trained on. Probable and present are different things, and nothing in the output distinguishes a column absorbed from documentation from one assembled because the parts looked right.

Microsoft security schemas make this sharper than most. Retired table names carry more weight in the training corpus than the ones that replaced them, tables get renamed mid-year, documentation lags the portal, and preview tables ship with no published column reference at all.

The fix is not a better prompt. It is refusing to let the model answer until it has checked.

## How to actually use this

These are not programs and there is nothing to run. Each one is **standing context** — text the
model reads before it answers. You put the text where your platform looks for it, then ask your
normal question. Nothing to invoke: type *"write me a KQL query for failed sign-ins"* and the
rules have already shaped the reply.

Two of the four routes are a file copy. The other two are a short build.

### Claude — copy one file

```
~/.claude/skills/kql-grounded-queries/SKILL.md
```

Copy [`skills/kql-grounding-agent/claude/SKILL.md`](skills/kql-grounding-agent/claude/SKILL.md)
there. **The folder name must be `kql-grounded-queries`** to match the skill's frontmatter. Claude
loads it automatically and triggers on anything KQL-shaped — including questions that never use
the word "KQL", like *"find agents on laptops in Defender"*.

On claude.ai or Claude Desktop rather than the CLI, paste the file's contents into a Project's
custom instructions instead. Same rules, slightly less automatic.

### GitHub Copilot — copy one file

```
.github/copilot-instructions.md        in a repository
%USERPROFILE%/copilot-instructions.md  for everything you do
```

Copy
[`skills/kql-grounding-agent/github-copilot/copilot-instructions.md`](skills/kql-grounding-agent/github-copilot/copilot-instructions.md).
It applies to every Copilot Chat conversation in that scope, with nothing to trigger. In SSMS and
some other hosts the repository-level file needs enabling once in Copilot Chat options.

### Copilot Studio — build an agent (about twenty minutes)

Not a file copy. You create an agent, paste the supplied instructions into it, attach the
Microsoft Learn Docs and KQL Search MCP servers as tools, and publish it. Your users then talk to
*that agent* rather than to generic Copilot. Full steps, and the two settings that silently break
it, are in [`skills/kql-grounding-agent/copilot-studio/`](skills/kql-grounding-agent/copilot-studio).

### Microsoft 365 Copilot — build a declarative agent

Also a build. Agent Builder or the Agents Toolkit, with the instructions pasted in, web search
scoped to the documentation, and ideally a Learn MCP plugin attached. Read the limitation note
first — without that plugin this version searches rather than fetches, which is a weaker promise.
Steps in [`skills/kql-grounding-agent/m365-copilot/`](skills/kql-grounding-agent/m365-copilot).

### One thing people miss

On Claude and Copilot Studio the **MCP servers are a separate setup step**. Connect
[KQL Search](https://www.kqlsearch.com/) and the Microsoft Learn Docs MCP server, or the agent
falls back to plain web fetches: schema verification mostly still works, but the prior-art step
gets thin. Each skill's README lists what to connect.

---

Each skill folder leads with the method — the ordered steps and the checks that make it work —
then offers that method as a file per platform. The Claude version is canonical in every case,
since it is the one used on real work; other platform versions carry the same rules adapted to
how that platform takes standing instructions, and say so where they have not been validated to
the same standard.

## Posts

Longer write-ups behind this work. Published versions may have moved on from these copies.

- [The KQL agent that refuses to guess](posts/the-kql-agent-that-refuses-to-guess.md) — how the grounding skill works and where it still falls over
- [Microsoft licensing for AI and agent security](posts/microsoft-licensing-for-ai-and-agent-security.md) — three billing meters, and what each licence actually turns on · [published](https://shashankraina.substack.com/p/microsoft-licensing-for-ai-and-agent)
- [Counting agents is harder than it should be](posts/counting-agents-is-harder-than-it-should-be.md) — why four screens give four different inventory numbers · [published](https://shashankraina.substack.com/p/microsoft-security-for-agents)

## Related

[aiagentsecurity.guide](https://aiagentsecurity.guide) — an interactive licence-to-capability grid across the five classes of AI asset, and an agent telemetry map showing where each agent type's activity lands and what gates it.

## Credit

The prior-art half of the KQL skill runs on [KQL Search](https://www.kqlsearch.com/), built by [Ugur Koc](https://ugurkoc.de/) ([@ugurkocde](https://github.com/ugurkocde)), who also put an MCP server on top of it so agents can search the corpus directly.

## Licence

MIT. See [LICENSE](LICENSE).

---

*Independent work. Not affiliated with or endorsed by Microsoft. Verify current product behaviour against Microsoft documentation before relying on anything here.*
