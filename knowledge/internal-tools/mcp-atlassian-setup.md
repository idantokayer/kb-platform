---
title: Atlassian MCP (Jira + Confluence) — Setup Guide
description: Connect your AI agent to Jira and Confluence via Atlassian's official Remote MCP Server, using the Streamable HTTP transport (the deprecated SSE endpoint is removed after 30 Jun 2026).
tags: [plm, internal-tools, mcp, atlassian, jira, confluence, setup]
---

# Atlassian MCP — Setup Guide

Connect your AI agent to Jira and Confluence for issue management, page editing, search, and more.

Part of [MCP Remote Tools](./mcp-remote-tools.md).

**Provider:** [Atlassian Remote MCP Server](https://mcp.atlassian.com) (official, hosted by Atlassian)

## Quick Start

### 1. Install mcp-remote (one-time)

```shell
npm install -g mcp-remote
```

### 2. Add to your agent config

```json
"atlassian": {
  "command": "mcp-remote",
  "args": [
    "https://mcp.atlassian.com/v1/mcp",
    "--transport",
    "http-first"
  ],
  "timeout": 120000
}
```

> **Endpoint:** use the **Streamable HTTP** endpoint `https://mcp.atlassian.com/v1/mcp` with `--transport http-first`. The old HTTP+SSE endpoint (`/v1/sse`) is **deprecated** — see Transport Migration below.
>
> **Valid JSON:** don't leave a trailing comma after the last item in `args` — some agent configs reject it as invalid JSON.

### 3. Authenticate

Restart your agent. A browser opens for Atlassian OAuth — log in with your Toluna Atlassian account. Grant the requested permissions. Tokens refresh automatically after that.

## What You Get

30+ tools covering Jira and Confluence:

**Jira:** Search issues (JQL); get/create/edit issues; transition issue status; add comments and worklogs; link issues; manage projects and issue types.

**Confluence:** Search pages (CQL); get/create/update pages and blog posts; manage comments (footer and inline); list spaces and page hierarchies.

## Authentication

- **OAuth via Atlassian** — uses Atlassian's own OAuth server (not our gateway)
- **First use:** Browser opens to `auth.atlassian.com` for consent
- **Subsequent uses:** Token auto-refreshes (offline_access scope)
- **Scopes:** openid, email, profile, offline_access
- **Migrating from SSE:** all existing access and admin controls carry over to the Streamable HTTP endpoint — no re-authorization needed beyond the normal OAuth flow

## Transport Migration (HTTP+SSE → Streamable HTTP)

Atlassian is **deprecating** the HTTP+SSE transport in favor of Streamable HTTP (for reliability and to align with the MCP spec).

|  | Old (deprecated) | New (use this) |
| --- | --- | --- |
| Endpoint | `https://mcp.atlassian.com/v1/sse` | `https://mcp.atlassian.com/v1/mcp` |
| Transport | HTTP+SSE | Streamable HTTP |
| Status | Backward-compat only, **removed after 30 June 2026** | Current, actively maintained |

**To migrate:** change the URL in your `args` to `/v1/mcp` and add `"--transport", "http-first"`, then restart your agent. Access/admin controls are unchanged.

> Verified working at Toluna (2026-08-31): repointed to `/v1/mcp`, reconnected, Confluence + Jira tools load and operate normally on the new transport.

Reference: [Atlassian HTTP+SSE Deprecation Notice](https://community.atlassian.com/forums/Atlassian-Remote-MCP-Server/HTTP-SSE-Deprecation-Notice/ba-p/3205484).

## Notes

- **Remote MCP** — runs on Atlassian's cloud, no local server needed
- **Read-only by default** — write operations (create/edit) require explicit tool approval in your agent config
- **Rate limits** — Atlassian enforces API rate limits; the MCP server handles backoff automatically

## Troubleshooting

| Issue | Fix |
| --- | --- |
| Tools not loading | Ensure `mcp-remote` is v0.1.29+: `npm list -g mcp-remote` |
| Still seeing SSE deprecation notices in tool output | You're still on `/v1/sse` — switch to the `/v1/mcp` + `--transport http-first` config above and restart |
| "connection closed" after working | Clear cache: `rm -rf ~/.mcp-auth` and restart |
| Invalid JSON / config won't parse | Remove any trailing comma after the last item in `args` |
| OAuth consent page won't load | Check you're logged into the correct Atlassian account |
| Permission errors on write operations | Verify your Atlassian account has edit permissions in the target project/space |

## Support

Platform Community Channel on Teams. Atlassian MCP docs: [mcp.atlassian.com](https://mcp.atlassian.com)
