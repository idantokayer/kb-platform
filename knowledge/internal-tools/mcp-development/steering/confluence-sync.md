---
title: Confluence Sync
description: The project pointer for the MCP Development Confluence-as-Git sync — remote location, the confluence-sync config/index page ID, project type, and naming/format conventions.
tags: [plm, mcp, steering, confluence-sync, pointer]
---

# Confluence Sync — MCP Development Project (remote pointer)

This is the **project pointer** (the `.git/config` analogue) for the MCP Development project under the Project Management Protocol (Confluence-as-Git). The high-churn **index** (local file → page ID) lives on the [confluence-sync](../confluence-sync.md) page.

Part of [MCP Development](../index.md) (Steering).

> **Two-artifact model.** This file holds only the low-churn pointer: remote location, the `confluence-sync` config-page ID, and conventions. The high-churn index lives on the `confluence-sync` page in Confluence so structural edits get real revision history. To add/move/rename a page, `register` it on that page (pull → merge your row → push).

## Remote Pointer

| Field | Value |
| --- | --- |
| **Cloud site** | `toluna.atlassian.net` |
| **Space** | SOA (key: `SOA`, ID: `842727492`) |
| **Main page** (remote root) | Internal Tools > MCP Development — see [MCP Development](../index.md) (remote ID `2768109569`) |
| **Main page URL** (remote pointer data) | `https://toluna.atlassian.net/wiki/spaces/SOA/pages/2768109569` |
| `confluence-sync` config/index page | ID `2864513039` — directly under the main page; its body IS the index |

## Project Type

**Single project** — one main page, one space (SOA). The meta-project / superproject model does not apply here.

## Conventions

### Page Naming

- Server pages: `{Name} MCP Server` (e.g., "Datadog MCP Server")
- Doc pages: plain descriptive title (e.g., "Remote MCP Gateway")
- No emojis in page titles (folders may have them)

### Content Format

- **Markdown by default** (`contentFormat: markdown`).
- Use `html` only when a page needs it (e.g., Mermaid diagrams) and note it on that page.

### Structural Folders

Structural folders (`docs/`, `Steering/`, `Servers/`, `skills/`) are mapped **on the** [confluence-sync](../confluence-sync.md) index page (ID `2864513039`) as first-class objects, alongside the file rows. Do not duplicate folder IDs here.

### When Creating a New Server Page

1. Create locally in `servers/{name}.md`.
2. `push` to Confluence under the `Servers/` folder.
3. `register` the new page on the `confluence-sync` index page — pull → add the row → push.
4. Update the main MCP Development page's server table with the new page link.
5. Update the public user guide (SOA, ID `2755264514`) if the server is live.

### Additional Sync Targets

Cross-space or externally-owned pages this project touches are listed under "Additional Sync Targets" on the `confluence-sync` index page — including the **Chamberlain MCP Design Document** (DD space, ID `2762047490`), which is **owned by the Chamberlain team: comment only, do not edit.**
