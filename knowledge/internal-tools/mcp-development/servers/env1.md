---
title: ENV1 MCP Server
description: The ENV1 MCP server — exposes Toluna's private Terraform registry and S3-backed service config store as MCP tools (8 tools), with per-profile Cognito auth for the S3 tools.
tags: [plm, mcp, server, env1, terraform-registry, config]
---

# ENV1 MCP Server

Part of [MCP Development → Servers](../index.md). The underlying platform is [Env1](../../env1.md). Architecture: [ENV1 MCP Server — Design](./env1-design.md).

| Field | Value |
| --- | --- |
| **Repo** | [toluna-env1-mcp-server](https://bitbucket.org/tolunaengineering/toluna-env1-mcp-server) |
| **Package** | `env1-mcp-server-py` v0.2.9 (CodeArtifact: `platform-pypi`) |
| **Python package** | `env1_mcp_server` |
| **Layer** | env1-mcp-server-layer (VPC-enabled) |
| **Profile** | `env1` → `mcp.toluna.me/plm/env1` |
| **Auth pattern** | Pattern A (token provider — ENV1's own Cognito) |

## Overview

The ENV1 MCP server exposes Toluna's ENV1 platform to AI agents. ENV1 is both a **private Terraform module registry** and a **service configuration store** (S3-backed). The server wraps the ENV1 HTTP API (`https://env1.toluna.me`) behind a set of MCP tools.

It follows the standard Toluna MCP package layout, runs both locally (stdio, browser OAuth) and remotely (AgentCore gateway, DDB-backed token), and ships the standard contract test.

## Tools

| Tool | Auth | Description |
| --- | --- | --- |
| `get-version` | No | Return the running server version |
| `list-all-registry-modules` | No | List Terraform modules by namespace (default `Prod`) |
| `get-module-versions` | No | All versions of a module |
| `get-module-info` | No | Module details (README, inputs, outputs, dependencies) |
| `read-s3-json` | Yes | Read service config from S3 (all configs, or a single type) |
| `write-s3-json` | Yes | Write/merge service config (deep merge) |
| `get-service-by-repo` | No | Resolve which service + tribe a repository maps to |
| `get-env-colors` | No | Current blue/green workspace color per environment |

`get-service-by-repo` and `get-env-colors` are unauthenticated GETs (the backend requires no auth for these endpoints — changed in v0.2.9).

## Namespaces

ENV1 = private Terraform module registry. Namespaces map to Bitbucket projects: `Prod` (most modules, default), `DataLab`, `ML-Prod`.

## Per-Profile Cognito Auth Delegation

The S3 tools (`read-s3-json`, `write-s3-json`) authenticate via ENV1's own Cognito pool — ENV1 was the first MCP to use this per-profile auth pattern:

1. OAuth server redirects to ENV1's Cognito (`env1-prod.auth.us-east-1.amazoncognito.com`)
2. Same Azure AD SSO (different pool, same IdP)
3. ENV1's pre-token Lambda adds a custom `role` claim
4. Token stored in DynamoDB (`session:{email}:env1`) for the remote gateway; `~/.mcp-auth/env1-auth/token.json` for local stdio
5. Handler reads the token and calls ENV1 S3 endpoints with `Authorization: Bearer {id_token}`

**Cognito:** Pool `us-east-1_kB6XoEgW8`, Client `1sue55v28rkb6cnrsoshr42u5l`, IdP `AzureAD`. The client performs an automatic **401 retry** (re-fetch token once, then `PermissionError`).

## S3 Config Semantics

**Read (`read-s3-json`):** `key = "servicename"` → all config files keyed by config name (`app`, `data`, `shared`, …); `key = "servicename/app"` → just that config. Resolution: list `s3_storage/list`, match by `{service}/` prefix, derive the tribe from the bucket name (`toluna-{tribe}-tf-config-data`), read `s3_storage/{tribe}/{service}/{type}-env.json`.

**Write (`write-s3-json`):** requires an explicit config type (refuses a bare service name); performs a **deep merge** (only provided keys updated); always reads existing config first, merges, writes back. For v2 (decoupled) services the write becomes a pending change — see the [Design page](./env1-design.md).

## Terraform Registry Internals

Bitbucket integration lives inside the ENV1 API, not the MCP: single OAuth consumer (`client_credentials`); token cache 55 min, response cache 120s; peak ~200–500 BB calls/hour (under the 1,000 RPH limit). The MCP does **not** talk to Bitbucket directly — it goes through ENV1's cached API, so adding the MCP does not increase Bitbucket pressure.

## Operational Notes

- **VPC required** — `env1.toluna.me` is private (account `654654530855`)
- First-time S3 auth (local): browser SSO popup (one-time), token cached and auto-refreshed
- Registry tools, `get-service-by-repo`, and `get-env-colors` need no auth
- Test locally: `MOCK_MODE=true python3 -m pytest tests/` (21 tests)

## Related

- [MCP Development Guide](../docs/mcp-development-guide.md)
- Child page: [ENV1 MCP Server — Design](./env1-design.md) (architecture, request flow, contract, v2 apply flow)
