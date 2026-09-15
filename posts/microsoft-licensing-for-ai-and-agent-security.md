# Microsoft licensing for AI and agent security

*Three billing meters, five classes of AI asset, and what each licence turns on.*

Published: [https://shashankraina.substack.com/p/microsoft-licensing-for-ai-and-agent](https://shashankraina.substack.com/p/microsoft-licensing-for-ai-and-agent)

---

Can I find local agents with E5? How do I find the agents in my environment at all? What do I need to stop my agents being fed malicious prompts? Do I need Agent 365 just to block one? Does any of this cover the Foundry projects sitting in Azure?

And many, many more. The questions keep changing. The answer is still the same: it depends!!

If you have followed how Microsoft does licensing over the years, that will not shock you. AI and agent security has managed to make it more complex again, if that was even possible.

All of those come down to licensing. I got tired of drawing the same diagram on whiteboards, so I built it properly. There's an interactive grid at [aiagentsecurity.guide/licensing](https://aiagentsecurity.guide/licensing.html): tick the licences your tenant holds and it lights up what you can see, govern, protect and detect across the five classes of AI asset. Everything else greys out, with the licence you would need written on it.

## Three billing meters

Microsoft bills AI security on three meters.

Microsoft 365 is per user — Copilot audit, the Purview stack, Defender XDR, and the agent control plane in Agent 365 and E7.

Azure is per resource and per token scanned — the Defender for Cloud plans covering Foundry and Azure OpenAI workloads.

Sentinel is per gigabyte ingested and retained, with the data lake tier priced separately again.

Money spent on one meter buys nothing on the others, and that is behind most of the expensive planning mistakes I see, because the meters don't line up with the org chart. The team that bought E5 is usually not the team that owns the Azure subscription where the Foundry projects run.

Select E5 in the grid and 21 of the 48 capabilities light up. Class 2, AI platform and workloads, stays almost entirely dark, because that whole class is billed on the Azure meter. Even E7 only reaches 43 of 48. The five it misses are Defender for Cloud posture, Defender for AI Services alerts, the APIM gateway logs and the two Sentinel data lake tables. No Microsoft 365 SKU covers them.

## What you get without Agent 365

I had this wrong for a while and had to go back and fix my own slides.

The Agent 365 service description has a base column covering Microsoft 365 Enterprise, Business, Education and Frontline plans. Four capabilities sit in it: inventory of agents in the Agent Registry, the basic governance actions (publish, deploy, block, delete, approve, assign, reassign owner, pin), conditions-based lifecycle rules, and registry sync, which pulls agents and their metadata in from external platforms.

So discovering agents built outside Microsoft, listing them and blocking them doesn't need the agent security licence. There's a catch that is easy to miss when you read the table row by row: all four are Microsoft 365 admin centre capabilities, and the Graph API row for the registry sits in the E7 and Agent 365 column. You can do these things by hand. You can't script them.

The same split runs through the whole grid. Seeing things is usually included in what you already own, and governing or detecting them is usually the paid tier. Agent identities are readable on any Entra tier; applying Conditional Access to them needs Agent 365.

## What E7 adds over E5

E7 is four things in one SKU: E5, Microsoft Copilot, Agent 365 and the Entra Suite. The Suite is the part that gets qualified least well.

It is five products — Private Access, Internet Access, ID Governance, ID Protection and Verified ID — and two of them overlap what an E5 tenant already owns. Entra ID P2 sits inside E5, and ID Protection is a P2 feature, so that one is not new value at all. ID Governance is a partial overlap: P2 already carries PIM, access reviews and the original entitlement management capabilities, and what the Suite adds on top is Lifecycle Workflows and the machine-learning-assisted review layer. Worth knowing before anyone prices the Suite as five new things.

The one that changes what you can do about AI is Entra Internet Access, a secure web gateway on the network path to public AI sites. It reaches traffic that Defender for Cloud Apps session policies never see. That is worth its own post.

## Empty tables are usually licence gates

`AgentsInfo` holds two populations under two licences. Local agents — coding assistants, desktop AI apps, local model runners — populate on Defender for Endpoint Plan 2, which is in E5, and discovery starts by itself once a device qualifies. Cloud agents in the same table need Agent 365. Most planning treats the cloud estate as the visible part; in practice the laptops show up first, and they are where I usually find ungoverned MCP servers.

`CloudAppEvents` works the same way. Copilot jailbreak and prompt-injection verdicts arrive on E5. The Agent 365 runtime rows — `InvokeAgent`, `InferenceCall`, `ExecuteToolBy*` — do not exist until Agent 365 instruments an agent. Same table, same query surface, different licence.

Before declaring a data gap, check the licence gate, then the query surface, then whether there is genuinely no activity. All three look identical from where you are sitting.

## The June purchase prerequisite

Since 1 June 2026, buying Agent 365 standalone requires a qualifying base plan: E5 for enterprise, F5-level Defender and Purview suites for frontline workers, Business Premium for SMB. E7 already includes Agent 365.

A second prerequisite causes confusion. Microsoft's Entra governance documentation states that an Agent 365 subscription needs a product carrying the `AAD_PREMIUM` service plan, which both Entra ID P1 and Microsoft 365 E3 satisfy. One rule is about what you are allowed to buy. The other is the technical dependency.

## How to use it

I tried writing all of this up as a table several times and it never worked — the content has three dimensions and a table flattens one of them. Ticking what you hold turned out to be the only version people could read.

A lit capability means the licence entitles you to it. Whether it is deployed, configured, and pointed at the agents you think it covers is a separate question. And don't quote a price from a web page, including mine. The split moves; it has moved twice while I have been maintaining the page.

The grid is at [aiagentsecurity.guide/licensing](https://aiagentsecurity.guide/licensing.html), free and without a sign-up. Its companion is the [agent telemetry map](https://aiagentsecurity.guide/agent-telemetry-map.html), which does the same job for telemetry: nine agent activity sources, the tables their activity lands in, and the licence gate on every one.

*Product behaviour described is current as of mid-September 2026 and reflects the Agent 365 service description and Microsoft Learn at that date. Verify against current documentation and your own tenant before making purchasing decisions.*
