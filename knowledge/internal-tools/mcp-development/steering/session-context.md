---
title: Session Context
description: Project status and onboarding reference for the MCP platform at mcp.toluna.me — live profiles and versions, repos, collaborator setup, recent changes, and active work.
tags: [plm, mcp, steering, session-context, status]
---

# MCP Development — Session Context

**Last Updated:** August 24, 2026

Project status and onboarding reference for the MCP platform at `mcp.toluna.me`. Part of [MCP Development](../index.md) (Steering).

## What's Live

| Profile | Server | Version | Tools |
| --- | --- | --- | --- |
| `datadog` | datadog-mcp-server-py | 0.7.5 | search-logs, aggregate-logs, search-traces, aggregate-traces, get-version |
| `usain-bolt` | usain-bolt-mcp-server-py | 0.1.6 | list-reports, get-report, get-test-logs, get-environment-health, execute-run, get-available-tags, get-version |
| `env1` | env1-mcp-server | 0.2.4 | list-all-registry-modules, get-module-versions, get-module-info, read-s3-json, write-s3-json, get-version |
| `chamberlain` | chamberlain-mcp-server-py | 0.1.2 | get-versions, compare-environments, get-deployment-history, get-deployment-status, list-products, list-env-syncs, get-version |
| `plm-kb` | Bedrock KB connector (SOA + AUT spaces) | — | Retrieve, AgenticRetrieveStream |
| `plm-internal-kb` | Bedrock KB connector (PLM space) | — | Retrieve, AgenticRetrieveStream |
| `devops-internal-kb` | Bedrock KB connector (DCI space) | — | Retrieve, AgenticRetrieveStream |

All accessible via `mcp-remote https://mcp.toluna.me/plm/{profile}` with Azure AD SSO.

## Repos

| Repo | Clone |
| --- | --- |
| remote-mcp-gateway | `git clone git@bitbucket.org:tolunaengineering/remote-mcp-gateway.git` |
| datadog-mcp-server-py | `git clone git@bitbucket.org:tolunaengineering/datadog-mcp-server.git` |
| usain-bolt-mcp-server-py | `git clone git@bitbucket.org:tolunaengineering/usain-bolt-mcp-server-py.git` |
| toluna-env1-mcp-server-py | `git clone git@bitbucket.org:tolunaengineering/toluna-env1-mcp-server.git` |
| chamberlain-mcp-server-py | `git clone git@bitbucket.org:tolunaengineering/chamberlain-mcp-server-py.git` |

## Getting Started as a Collaborator

1. Clone the repos you need (above)
2. Set up your local project folder: `~/kiro-sessions/mcp-development/`
3. Copy steering files into `.kiro/steering/` (auto-loaded by Kiro)
4. Set up global steering
5. Read the [MCP Development Guide](../docs/mcp-development-guide.md) for architecture
6. Read the server page for your target MCP (under Servers/)

## Recent Changes

### August 24, 2026

- **devops-internal-kb deployed** — new Bedrock KB (`JWDPJCK3HH`) indexing DCI space (DevOps), exposed as `devops-internal-kb` profile
- ENV1 MCP v0.2.4 — injectable TokenProvider, token refresh, config resolution fix, consistent error shapes
- Third KB brings the platform to 7 live profiles

### July 29, 2026

- PLM Internal KB deployed (`plm-internal-kb`); per-profile KB tool filtering in proxy; reorganized steering

### July 28, 2026

- Chamberlain MCP deployed (Phase 1, 6 read-only tools); ENV1 MCP deployed with per-profile Cognito auth

## Active Work

| Item | Owner | Status |
| --- | --- | --- |
| ENV1 v2 config service gate support | Eli + ENV1 team | Open (contract TBD) |
| Usain Bolt PR #8 (tool redesign) — fix failing tests | Alina | Pending |
| Usain Bolt PR #5 (get-report parallelization) | Alina | Approved, pending merge |
| Public MCP Gateway (API key auth for clients) | Alon + Platform | Design approved |
| V2 Gateway: Lambda Function URLs (DYNAMIC targets) | Platform | Planned |

## Documentation Map

| What | Where |
| --- | --- |
| How to build an MCP | [MCP Development Guide](../docs/mcp-development-guide.md) |
| Gateway architecture | [Remote MCP Gateway](../docs/remote-mcp-gateway.md) |
| Standards & conventions | [MCP Development Protocol](./mcp-development-protocol.md) |
| Operational runbook | [MCP Platform Operations](./mcp-platform-operations.md) |
| KB onboarding runbook | [Confluence Platform KB](../servers/confluence-kb.md) |
| Server details | Servers/ folder (one page per server) |
| End-user setup | [MCP Remote Tools — User Guide](../../mcp-remote-tools.md) |
