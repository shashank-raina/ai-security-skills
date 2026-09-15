# AI Security Skills

Agent skills for working on Microsoft AI and agent security, built around one idea: an agent should verify what it claims rather than producing something that looks right.

Everything here is used on real work. The skills are plain Markdown instruction files — no framework, no dependencies.

---

## Skills

| Skill | What it does | Platforms |
|---|---|---|
| [**kql-grounding-agent**](skills/kql-grounding-agent) | Writes KQL for Sentinel, Log Analytics and Defender XDR without inventing table or column names. Verifies every identifier against Microsoft documentation before composing anything, and stops instead of guessing when it cannot. | Claude · Copilot Studio · GitHub Copilot · M365 Copilot |

More will land here as they prove themselves in use.

## Why grounding

A language model holds no copy of the Defender XDR schema. It has weights encoding which tokens follow which other tokens, so in a query beginning `AgentsInfo | where`, names like `TimeGenerated`, `DeviceName` and `AccountName` are all highly probable because they appear constantly in the KQL it trained on. Probable and present are different things, and nothing in the output distinguishes a column absorbed from documentation from one assembled because the parts looked right.

Microsoft security schemas make this sharper than most. Retired table names carry more weight in the training corpus than the ones that replaced them, tables get renamed mid-year, documentation lags the portal, and preview tables ship with no published column reference at all.

The fix is not a better prompt. It is refusing to let the model answer until it has checked.

## Using these

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
