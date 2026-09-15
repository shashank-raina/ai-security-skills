# GitHub Copilot

Same rules as the Claude skill, written as a custom instructions file.

**File:** [`copilot-instructions.md`](copilot-instructions.md) · **Method:** [`../README.md`](../README.md)

> Adapted, **not yet validated**. The file format follows Microsoft's documented custom-instructions
> feature, but the behaviour it produces has not been tested the way the Claude version has. Watch
> the first few answers before trusting it.

## Install

Pick the scope you want:

```
.github/copilot-instructions.md          one repository
%USERPROFILE%/copilot-instructions.md    everything you do, everywhere
```

Repository-level and user-level instructions apply together, so a shared repo standard and your
own preferences coexist.

In some hosts the repository file needs enabling once — in SQL Server Management Studio, for
example, it is **Tools → Options → GitHub → Copilot → Copilot Chat**, "Enable custom instructions
to be loaded from .github/copilot-instructions.md files".

## Triggering

Nothing to invoke. It applies to every Copilot Chat conversation in scope. Custom instructions are
not shown in the chat view, but when Copilot uses the file it lists it in the response's
References.

## Prior art

The instructions point at [KQL Search](https://www.kqlsearch.com/) for published queries. If your
Copilot setup supports MCP servers, attaching KQL Search turns that step into a tool call instead
of a web search, which is a meaningful upgrade.

## Checking it is working

Ask for a query and look at the References list on the answer. If `copilot-instructions.md` is not
there, the file is not being loaded — check the path and, on hosts that need it, the option above.
