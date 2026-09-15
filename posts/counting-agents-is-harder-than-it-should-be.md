# Counting agents is harder than it should be

*Notes on Microsoft agent telemetry: why four different screens give you four different inventory numbers, and how to read an empty table.*

Both the inspiration for this post and its end result are the same thing: the [Agent Telemetry Map](https://aiagentsecurity.guide/agent-telemetry-map.html) — a free interactive map of nine agent activity sources, the sixteen tables and destinations their telemetry lands in, and the licence gate on every edge. What follows is what building it taught me.

Published: [https://shashankraina.substack.com/p/microsoft-security-for-agents](https://shashankraina.substack.com/p/microsoft-security-for-agents)

---

Earlier this summer I was in a workshop with a customer's platform team and asked what I thought was a warm-up question: how many AI agents are running in this tenant? We spent most of the afternoon on it and ended up with four numbers from Microsoft's own screens. The Agent Registry had the biggest count. Entra had a smaller one. Defender's inventory had a different one again, and it was the only screen that knew about the coding assistants on developer laptops. A Graph API call gave us a fourth.

All four numbers are correct. Each screen answers a different question. The registry counts published titles — things that could run, whether or not anyone uses them. Defender counts what security tooling can see. Entra counts what holds a directory principal, meaning what you can actually govern. The runtime tables count what ran.

These days I put the question itself in front of the customer, so that they understand what they are seeing and how to interpret those numbers. Agents in the registry with no Entra principal are the ungoverned population. Agents Defender sees on laptops that appear nowhere else are an endpoint estate that probably isn't on any risk register. An Entra identity with no telemetry means nobody instrumented observability for it.

These notes and the website are just my way of learning, and I hope they help others too.

The rest of this post covers the parts I keep having to explain in person, plus a mistake I made along the way.

## Empty tables

When an agent-related query comes back empty, there are three possible reasons: the licence gate isn't met, you're on the wrong query surface, or there's genuinely no data. They look identical and need different fixes.

The query surfaces are the most confusing part. This telemetry is spread across Log Analytics, Advanced Hunting, the Sentinel data lake, and a few things you can only reach through Graph or a portal. Tables don't carry over between them. `CopilotActivity` is a Log Analytics table; Advanced Hunting has never heard of it. `UnifiedAgentObservability` only exists in the data lake. On Advanced Hunting and the lake, querying a table that isn't in your tenant's schema returns a hard 400 rather than an empty result, and the `union isfuzzy=true` pattern people use to soften missing tables only works in Log Analytics.

Timestamps are a smaller version of the same problem. Advanced Hunting uses `Timestamp`, Log Analytics uses `TimeGenerated`, and tables streamed to both carry both columns. If data arrives more than 48 hours late, ingestion overwrites `TimeGenerated` with the arrival time — I once spent a morning investigating a gap in a timeline that turned out to be a late batch stamped with the wrong day.

The licence gates don't run in the direction you'd expect. `AgentsInfo`, the closest thing to a fleet inventory table, fills its local-agent rows on Defender for Endpoint P2 — included in Microsoft 365 E5, so E5 tenants have this without buying anything — but needs Agent 365 for cloud agents. So you can see the agents on laptops before you can see the ones in the cloud.

## A mistake I shipped

`AgentsInfo` stores repeated snapshots of every agent, so before counting anything you have to reduce it to latest state per agent. I wrote this:

```
AgentsInfo
| summarize arg_max(Timestamp, *)
```

and put it in a workbook tile. Without the `by AgentId`, that returns one row — the most recently written row in the whole table. My inventory tile reported a fleet of one, and I didn't spot why for longer than I want to admit. The version that works:

```
AgentsInfo
| summarize arg_max(Timestamp, *) by AgentId
| where LifecycleStatus != "Deleted"
| summarize Agents = count() by Platform
```

The lifecycle filter matters too, otherwise deleted agents stay in your count. In fairness to the documentation, the `LifecycleStatus` column and its values are all in the schema reference, and Microsoft's own sample queries for the table use the correct `by AgentId` reduction. What the docs never state is that the table stores repeated snapshots per agent — so if you write your query from the column list alone, nothing warns you what a bare `arg_max` will do. Snapshot-style tables are turning up more often in this space and the same mistake is available in all of them.

## The "Copilot Studio" label in Entra

At one tenant I looked at recently, the Entra agent identity view showed a couple of thousand entries, all labelled Copilot Studio. The security team was alarmed, understandably — nobody had built two thousand agents in Copilot Studio. As far as anyone knew, nobody had built two hundred.

What's going on: every Copilot Studio app-based agent identity is created as a child of one shared, global blueprint, and that blueprint's integration covers more than the Studio. Power Automate agent flows get their Entra Agent IDs through the same route. There are also field reports of first-party product agents provisioned through Power Platform landing under it — I haven't found that documented anywhere, so check your own tenant before repeating it. The label on those identities records which machinery created them; who built the thing is a separate question the label doesn't answer.

Two practical consequences. Counting that blueprint's children as "Copilot Studio agents someone built" overcounts, sometimes badly. And any Conditional Access policy or disable action aimed at the blueprint hits everything under it, flows included.

Some agent populations don't show up in these views at all. Copilot Studio agents built before March 18, 2026 — the date the Studio started creating agent identities automatically — or before the tenant opted in run on ordinary app registrations or the maker's own credentials. The way I find them is the service principal tag `AgentCreatedBy:CopilotStudio`, which Microsoft's own migration guidance calls the most reliable discovery signal; a second tag, `power-virtual-agents-{agent-id}`, links each service principal back to the specific agent in Copilot Studio. Declarative agents built with the agent builder inside M365 Copilot don't get an Entra Agent ID — the surface an agent was created on decides whether it gets an identity, which surprised me when I first confirmed it. Custom Security Copilot agents currently appear in neither the Agent 365 registry nor Entra Agent ID (Microsoft has said both are being worked on); for now their tracks are in the unified audit log after an opt-in toggle, and their interactions reach `CloudAppEvents` via the Microsoft 365 connector.

So on today's product surface, the registry overcounts and the identity views undercount. A single number quoted in a steering meeting is missing part of the estate either way.

## Two habits

Don't quote an agent count without saying which screen produced it. And don't declare a data gap until you've checked the licence gate and the query surface. The table names have already churned several times this year; these two habits still apply.

The full picture — every source, every table, every gate — is on the [map](https://aiagentsecurity.guide/agent-telemetry-map.html), no sign-up, updated as the products change.

*The observations above come from real tenants, generalised — nothing here identifies anyone. Product behaviour described is current as of early September 2026. Verify against Microsoft Learn and your own tenant's schema tab before building on any of it.*
