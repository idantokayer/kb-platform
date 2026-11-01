---
title: MCP Remote Tools — AI Agent Configuration
description: Toluna's remote MCP platform — connect any AI agent to shared company tools (observability, testing, infra, deployments, KBs) through a single managed gateway with corporate SSO.
tags: [plm, internal-tools, mcp, ai-agents, gateway]
---

# MCP Remote Tools

Toluna's remote MCP (Model Context Protocol) platform. Connect any AI agent to shared company tools — observability, testing, infrastructure, deployments, and more — through a single managed gateway with corporate SSO.

Part of [Internal Tools](./index.md). Setup guides: [Bitbucket MCP](./mcp-bitbucket-setup.md), [Atlassian MCP](./mcp-atlassian-setup.md), [Google Docs MCP](./mcp-google-docs-setup.md). For building servers, see [MCP Development](./mcp-development/index.md).

**No API keys. No local dependencies. No VPN. Just add a config block and authenticate once.**

## What's Available

| MCP Profile | Domain | Tools | Repo |
| --- | --- | --- | --- |
| **datadog** | Observability — logs, traces, APM | search-logs, aggregate-logs, search-traces, aggregate-traces, correlate-traces, get-version | [datadog-mcp-server](https://bitbucket.org/tolunaengineering/datadog-mcp-server) |
| **usain-bolt** | E2E Testing — reports, execution, health | list-reports, get-report, get-test-logs, get-environment-health, execute-run, get-available-tags, compare-runs, get-failed-components, get-component-health, get-version | [usain-bolt-mcp-server-py](https://bitbucket.org/tolunaengineering/usain-bolt-mcp-server-py) |
| **env1** | Infrastructure — Terraform registry, service config | list-all-registry-modules, get-module-versions, get-module-info, read-s3-json, write-s3-json, get-version | [toluna-env1-mcp-server](https://bitbucket.org/tolunaengineering/toluna-env1-mcp-server) |
| **chamberlain** | Deployments — versions, drift, history, traits | get-versions, compare-environments, get-deployment-history, get-deployment-status, list-products, list-env-syncs, get-traits, compare-traits, get-version | [chamberlain-mcp-server-py](https://bitbucket.org/tolunaengineering/chamberlain-mcp-server-py) |
| **plm-kb** | Platform Knowledge Base — public docs (Platform Engineering + Test Automation) | Retrieve, AgenticRetrieveStream | (native KB connector) |
| **plm-internal-kb** | Platform Internal KB — internal engineering docs (PLM space) | Retrieve, AgenticRetrieveStream | (native KB connector) |
| **devops-internal-kb** | DevOps Internal KB — DevOps team docs (DCI space) | Retrieve, AgenticRetrieveStream | (native KB connector) |
| **aut-internal-kb** | Test Automation Internal KB — Test Automation team docs (ATP space) | Retrieve, AgenticRetrieveStream | (native KB connector) |

More profiles coming. Any team can add their tools — see "Adding Your Tools" below.

**Infrastructure repo:** [remote-mcp-gateway](https://bitbucket.org/tolunaengineering/remote-mcp-gateway)

### Knowledge Base Profiles

| Profile | Spaces Indexed | Audience |
| --- | --- | --- |
| `plm-kb` | SOA (Platform Engineering), AUT (Test Automation) | Public — Dev Platform Group |
| `plm-internal-kb` | PLM | Platform team internal |
| `devops-internal-kb` | DCI (DevOps) | DevOps team internal |
| `aut-internal-kb` | ATP (Test Automation) | Test Automation team internal |

To add your team's docs as a KB: create a dedicated Confluence space, then contact the Platform team. See the [Confluence Platform KB](./mcp-development/servers/confluence-kb.md) doc for the full onboarding runbook.

## Quick Start

### 1. Install mcp-remote (one-time)

```shell
npm install -g mcp-remote
```

### 2. Add to your agent config

```json
"datadog": {
  "command": "mcp-remote",
  "args": ["https://mcp.toluna.me/plm/datadog"],
  "timeout": 120000
},
"usain-bolt": {
  "command": "mcp-remote",
  "args": ["https://mcp.toluna.me/plm/usain-bolt"],
  "timeout": 120000
},
"env1": {
  "command": "mcp-remote",
  "args": ["https://mcp.toluna.me/plm/env1"],
  "timeout": 120000
},
"chamberlain": {
  "command": "mcp-remote",
  "args": ["https://mcp.toluna.me/plm/chamberlain"],
  "timeout": 120000
},
"plm-kb": {
  "command": "mcp-remote",
  "args": ["https://mcp.toluna.me/plm/plm-kb"],
  "timeout": 120000
},
"plm-internal-kb": {
  "command": "mcp-remote",
  "args": ["https://mcp.toluna.me/plm/plm-internal-kb"],
  "timeout": 120000
},
"devops-internal-kb": {
  "command": "mcp-remote",
  "args": ["https://mcp.toluna.me/plm/devops-internal-kb"],
  "timeout": 120000
},
"aut-internal-kb": {
  "command": "mcp-remote",
  "args": ["https://mcp.toluna.me/plm/aut-internal-kb"],
  "timeout": 120000
}
```

### 3. Authenticate

Restart your agent. Browser opens for Azure AD SSO. Log in once — tokens refresh automatically.

## How It Works

```
AI Agent -- stdio -- mcp-remote -- HTTPS -- mcp.toluna.me -- AWS Gateway -- Tool Backends
```

- **Profile filtering** — each URL returns only that profile's tools
- **Per-profile auth** — some profiles (env1) authenticate via their own Cognito pool for full RBAC
- **Tool name rewriting** — gateway names cleaned server-side

## Adding Your Tools

1. Build a Python package with `tool_definitions.py`, `client.py`, `server.py`
2. Publish to CodeArtifact
3. Platform team handles gateway integration

See the [MCP Development Guide](./mcp-development/docs/mcp-development-guide.md) for the full walkthrough.

## Troubleshooting

| Issue | Fix |
| --- | --- |
| Browser shows "redirect_mismatch" | `rm -rf ~/.mcp-auth` then restart agent |
| Tools not showing | Check config syntax, `npm list -g mcp-remote` |
| Timeout on first call | Lambda cold start (~2s). Retry. |
| "insufficient_scope" | `rm -rf ~/.mcp-auth` and restart |
| "connection closed: initialize response" | `rm -rf ~/.mcp-auth` and restart |
| ENV1 S3 tools "authentication required" | Authenticate once via ENV1's Cognito. Clear `~/.mcp-auth` and restart. |

## Requirements

- Node.js 18+ (for mcp-remote)
- Toluna corporate account (Azure AD SSO)

## Support

Platform Community Channel on Teams.
