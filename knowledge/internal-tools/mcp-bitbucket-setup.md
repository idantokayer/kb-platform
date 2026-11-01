---
title: Bitbucket MCP — Setup Guide
description: Connect your AI agent to Bitbucket Cloud for repository browsing, pull request management, and code search using a scoped Atlassian API token.
tags: [plm, internal-tools, mcp, bitbucket, setup]
---

# Bitbucket MCP — Setup Guide

Connect your AI agent to Bitbucket Cloud for repository browsing, pull request management, code search, and more.

Part of [MCP Remote Tools](./mcp-remote-tools.md).

**Package:** [@aashari/mcp-server-atlassian-bitbucket](https://github.com/aashari/mcp-server-atlassian-bitbucket) (open source, v2.3.0+)

## Quick Start

### 1. Install

```shell
npm install -g @aashari/mcp-server-atlassian-bitbucket
```

Requires Node.js 18+.

### 2. Create an API Token

Go to [Atlassian API Tokens](https://id.atlassian.com/manage-profile/security/api-tokens) and create a **Scoped API Token**:

1. Click **"Create API token with scopes"**
2. Select **Bitbucket** as the product
3. Name it (e.g., "AI Agent")
4. Select scopes (see table below)
5. Copy the token (starts with `ATATT`)

### 3. Required Scopes

| Scope | What it enables |
| --- | --- |
| **repository** | Read repos, branches, commits, file content |
| **workspace** | List workspaces |
| **pullrequest** | Read/write pull requests, comments, approvals |

**Minimum for read-only:** `repository` + `workspace`

**Full functionality (recommended):** `repository` + `workspace` + `pullrequest`

### 4. Add to Your Agent Config

Add to your MCP config file (e.g., `~/.kiro/settings/mcp.json`):

```json
"bitbucket": {
  "command": "mcp-atlassian-bitbucket",
  "args": [],
  "disabled": false,
  "env": {
    "ATLASSIAN_USER_EMAIL": "your.name@toluna.com",
    "ATLASSIAN_API_TOKEN": "ATATT3x...(your scoped token)",
    "ATLASSIAN_SITE_NAME": "bitbucket",
    "BITBUCKET_DEFAULT_WORKSPACE": "tolunaengineering"
  }
}
```

### 5. Restart your agent

The Bitbucket MCP should appear with 6 tools available.

## Available Tools

| Tool | Description |
| --- | --- |
| `bb_get` | Read any Bitbucket API endpoint (repos, PRs, branches, files) |
| `bb_post` | Create resources (PRs, comments, approve PRs) |
| `bb_put` | Replace resources (full updates) |
| `bb_patch` | Partial updates (PR title, description) |
| `bb_delete` | Delete resources (branches, decline PRs) |
| `bb_clone` | Clone a repository to local filesystem |

These are generic HTTP tools that map to the full [Bitbucket Cloud REST API](https://developer.atlassian.com/cloud/bitbucket/rest/).

## Common Operations

| What you want to do | Example |
| --- | --- |
| List repos | `bb_get(path: "/repositories/tolunaengineering")` |
| Get PR details | `bb_get(path: "/repositories/tolunaengineering/my-repo/pullrequests/42")` |
| List open PRs | `bb_get(path: "/repositories/tolunaengineering/my-repo/pullrequests", queryParams: {"state": "OPEN"})` |
| Get file content | `bb_get(path: "/repositories/tolunaengineering/my-repo/src/main/path/to/file.py")` |
| Create a PR | `bb_post(path: "/repositories/tolunaengineering/my-repo/pullrequests", body: {...})` |
| Approve a PR | `bb_post(path: "/repositories/tolunaengineering/my-repo/pullrequests/42/approve", body: {})` |

## Notes

- **No VPN required** — uses Bitbucket Cloud API (public)
- **Token never expires** — scoped API tokens are long-lived (until revoked)
- **Workspace:** Use `tolunaengineering` for all Toluna repos
- **TOON format:** Responses use a token-efficient format by default. Use `outputFormat: "json"` if you need standard JSON.
- **JMESPath filtering:** Use the `jq` parameter to extract only the fields you need and reduce token costs

## Troubleshooting

| Issue | Fix |
| --- | --- |
| "Authentication failed" / 403 | Token expired or wrong scopes. Regenerate at [Atlassian API Tokens](https://id.atlassian.com/manage-profile/security/api-tokens) |
| "Resource not found" / 404 | Check workspace slug and repo name (case-sensitive, use slugs from URL) |
| Tools not loading | Verify Node.js 18+ installed: `node --version` |
| Command not found | Reinstall: `npm install -g @aashari/mcp-server-atlassian-bitbucket` |

## Legacy: App Passwords (Deprecated)

Bitbucket App Passwords are being deprecated (June 2026). If you're still using them:

```json
"env": {
  "ATLASSIAN_BITBUCKET_USERNAME": "your_username",
  "ATLASSIAN_BITBUCKET_APP_PASSWORD": "your_app_password",
  "BITBUCKET_DEFAULT_WORKSPACE": "tolunaengineering"
}
```

**Required permissions:** Workspaces: Read, Repositories: Read+Write, Pull Requests: Read+Write

**Migrate to scoped API tokens** — they're more secure and future-proof.

## Support

Platform Community Channel on Teams. Package docs: [github.com/aashari/mcp-server-atlassian-bitbucket](https://github.com/aashari/mcp-server-atlassian-bitbucket)
