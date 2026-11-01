---
title: confluence-sync (Ref Registry)
description: The ref registry / tracking-map index for the MCP Development project — maps each local file and structural folder to its Confluence page/folder ID for the pull/push/register sync flow.
tags: [plm, mcp, confluence-sync, registry, index]
---

# confluence-sync (Ref Registry)

Part of [MCP Development](./index.md). The sync protocol itself is documented in [Confluence Sync](./steering/confluence-sync.md).

> **Ref registry for the MCP Development project.** This page IS the index (the git tracking-map analogue) for the Project Management Protocol (Confluence-as-Git). Its body maps each local file **and structural folder** to its Confluence page/folder ID. It is the **shared source of truth for IDs** — `pull`/`push`/`register` all read and converge on this page.
>
> **Maintenance:** every structural change (page/folder add/move/rename) MUST be `register`ed here — pull this page first, merge your row in, push back with a descriptive version message. **Merge rows; never overwrite the whole map.**

## Structural Folders (containers under the main page)

| Container | Confluence Folder ID | Notes |
| --- | --- | --- |
| `docs/` | `2767388711` | design docs, guides, gateway architecture |
| `Steering/` | `2767650835` | steering docs (protocol, ops, session-context, pointer) |
| `Servers/` | `2768568321` | one page per MCP server |
| `skills/` | `2860089365` | agent skills — no pages indexed yet; register each skill page here when created |

## Index — Local File → Confluence Page

| Local File | Confluence Location | Page ID | Title |
| --- | --- | --- | --- |
| `docs/mcp-development-guide.md` | docs/ | `2767028252` | MCP Development Guide |
| `docs/remote-mcp-gateway.md` | docs/ | `2767945754` | Remote MCP Gateway |
| `docs/mcp-gateway-v2-design.md` | docs/ | `2846982145` | MCP Gateway — Generic Handler & V2 |
| `docs/mcp-gateway-microservice-requirements.md` | docs/ | `2846195754` | MCP Gateway Microservice Requirements |
| `docs/mcp-gateway-m2m-headless-auth-design.md` | docs/ | `2880995365` | MCP Gateway — Headless / M2M Auth (Design) |
| `docs/mcp-design-alternatives.md` | docs/ | `2859925551` | Design Alternatives & Incidents |
| `.kiro/steering/session-context.md` | Steering/ | `2767028230` | Session Context |
| `.kiro/steering/mcp-development-protocol.md` | Steering/ | `2767290381` | MCP Development Protocol |
| `.kiro/steering/mcp-platform-operations.md` | Steering/ | `2776334338` | MCP Platform Operations |
| `.kiro/steering/confluence-sync.md` | Steering/ | `2776465409` | Confluence Sync (project pointer) |
| _(the index page itself)_ | under main page | `2864513039` | confluence-sync (this page — the ref registry) |
| `servers/datadog.md` | Servers/ | `2768371714` | Datadog MCP Server |
| `servers/usain-bolt.md` | Servers/ | `2767388687` | Usain Bolt MCP Server |
| `servers/env1.md` | Servers/ | `2767421467` | ENV1 MCP Server |
| `servers/chamberlain.md` | Servers/ | `2770829314` | Chamberlain MCP Server |
| `servers/google-docs.md` | Servers/ | `2768502786` | Google Docs MCP Server |
| `servers/confluence-kb.md` | Servers/ | `2768535553` | Confluence KB MCP Server |

## Additional Sync Targets

Pages this project syncs to that live outside the main-page tree, or are owned by another team.

| Page | Space | Page ID | Notes |
| --- | --- | --- | --- |
| MCP Remote Tools — AI Agent Configuration | SOA | `2755264514` | Public user-facing guide (kept in sync when a server goes live) |
| Chamberlain MCP Design Document | DD | `2762047490` | **Owned by the Chamberlain team — comment only, do NOT edit** |
