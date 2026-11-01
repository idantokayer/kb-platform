---
title: Chamberlain MCP Server
description: The Chamberlain MCP server — deployment versions, drift, history, and feature-flag traits (9 read tools) plus 3 write tools (deploy-sync, env-sync) with Pattern A Cognito auth.
tags: [plm, mcp, server, chamberlain, deployments]
---

# Chamberlain MCP Server

Part of [MCP Development → Servers](../index.md).

| Field | Value |
| --- | --- |
| **Repo** | [chamberlain-mcp-server-py](https://bitbucket.org/tolunaengineering/chamberlain-mcp-server-py) (branch: `chamberlain-mcp`) |
| **Package** | `chamberlain-mcp-server-py` v0.1.9 |
| **Profile** | `chamberlain` → `mcp.toluna.me/plm/chamberlain` |
| **Developer** | Yarin Hilleli |
| **Auth pattern** | Pattern A (token provider — Chamberlain's own Cognito) |

## Current State

Phase 2 (write tools) — fully live. v0.1.9 published to CodeArtifact; deployed to `mcp.toluna.me/plm/chamberlain`. Remote write-tool auth is live (`gw_hooks/chamberlain_mcp_server.py`); `execute-deploy-sync` TRN→STG confirmed HTTP 201 end-to-end (2026-09-01).

## Changelog

| Version | Date | Changes |
| --- | --- | --- |
| 0.1.0 | 2026-07-28 | Initial release — 6 read-only tools |
| 0.1.1 | 2026-07-29 | 11 code review items fixed |
| 0.1.2 | 2026-08-11 | Added `get-version` |
| 0.1.4 | 2026-08-20 | Added `get-traits` and `compare-traits` |
| 0.1.6 | 2026-08-30 | Phase 2 write tools; generic-handler migration; removed `cognito_token` from schemas (Pattern A); PROD guard moved to backend |
| 0.1.9 | 2026-09-01 | Remote write auth fully live — `gw_hooks/chamberlain_mcp_server.py` confirmed; `execute-deploy-sync` end-to-end 201 |

## Tools (12 total)

### Read-Only (9 — Phase 1)

`get-versions`, `compare-environments`, `get-deployment-history`, `get-deployment-status`, `list-products`, `list-env-syncs`, `get-traits`, `compare-traits`, `get-version`.

### Write (3 — Phase 2, requires Cognito auth)

| Tool | Description |
| --- | --- |
| `preview-deploy-sync` | Preview what would be deployed from source → target env. Returns plan + expiring `plan_id` (5-min TTL). Read-only — no auth required. |
| `execute-deploy-sync` | Execute a previewed deploy plan by `plan_id`. Requires Cognito auth. Always preview first. |
| `manage-env-sync` | Create or delete an auto-propagation rule between environments. Requires Cognito auth. |

## Architecture

Follows the **generic-handler model**: `tool_definitions.py` is the single source of truth (plain dicts with a `method` binding); the gateway's generic handler dispatches directly from `TOOLS`; the package is pure and runs identically local (stdio, `CHAMBERLAIN_TOKEN` env var) and remote (generic-handler Lambda, token injected by `gw_hooks/chamberlain_mcp_server.py` from the DynamoDB session store). See [MCP Gateway — Generic Handler & V2](../docs/mcp-gateway-v2-design.md).

### Auth (Pattern A)

Read tools: no auth (public endpoints). Write tools: Bearer token from Chamberlain's own Cognito pool (with a `role` claim from a pre-token-generation trigger). Local dev: `CHAMBERLAIN_TOKEN` env var. Remote: `gw_hooks/chamberlain_mcp_server.py` reads the claim-bearing JWT from the DynamoDB session store. Both service-team Cognito prerequisites are complete (callback URL + auth-code flow).

### Re-authentication (when session expires)

If write tools return "Chamberlain session expired or not found", delete the cached token file (`~/.mcp-auth/mcp-remote-0.1.37/...tokens.json`), toggle the `chamberlain` entry in `~/.kiro/settings/mcp.json` (add `"disabled": true`, save, remove, save), log in via the browser popup, then verify with a `preview-deploy-sync` → `execute-deploy-sync` (TRN→STG, SXTI-BE, `skip_older=false`) → expect HTTP 201.

## Write-Tool Safety

Two-phase preview → execute: `preview-deploy-sync` returns a concrete plan + `plan_id` (5-min TTL); `execute-deploy-sync` takes only the `plan_id`, re-validates TTL, commits. **Admin-only** (Cognito `admin` role claim, enforced by the backend — 403 otherwise). **PROD is blocked as a target** (backend 403). Caller must name exactly what is affected — no "all" default. See the [MCP Development Protocol](../steering/mcp-development-protocol.md).

## Key Design Decisions

- **No business logic in the MCP** — PROD guard, RBAC, telemetry, and user attribution are enforced by the Chamberlain API. The MCP is a thin protocol adapter.
- **Not FastMCP** (ADR-001) — uses the low-level `mcp` SDK directly. See [Design Alternatives & Incidents](../docs/mcp-design-alternatives.md).
- `in_sync` list dropped from `compare-environments` — returns count only.
- `json.dumps` never with `indent=2` — ~30% token waste.

## Operational Notes

- VPC required — `chamberlain.toluna.me` is a private API
- After updating the Lambda layer, Platform must restart `mcp-profile-filter` to bust the interceptor's `tools/list` cache
- Test write tools with `skip_older=false`, TRN→STG, SXTI-BE — guarantees `to_deploy: 1` so a 201 confirms auth
- Alfred fires two SNS notifications per deploy — two rows in deployment history is expected
- 40 products tracked, 5 environments
