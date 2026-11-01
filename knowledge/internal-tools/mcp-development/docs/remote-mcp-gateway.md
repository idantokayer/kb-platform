---
title: Remote MCP Gateway
description: Architecture of Toluna's remote MCP gateway at mcp.toluna.me — API Gateway OAuth server, domain proxy, AgentCore gateway with interceptor, and per-server handler Lambdas.
tags: [plm, mcp, gateway, architecture, oauth]
---

# Remote MCP Gateway

**Repo:** [remote-mcp-gateway](https://bitbucket.org/tolunaengineering/remote-mcp-gateway) · **Live Endpoint:** `mcp.toluna.me` · **AWS Account:** 457416912276 (datalab)

Part of [MCP Development](../index.md). User-facing config: [MCP Remote Tools](../../mcp-remote-tools.md).

---

## Architecture

```
AI Agent (mcp-remote https://mcp.toluna.me/plm/{profile})
  │ HTTPS (OAuth2 — Azure AD SSO)
  ▼
mcp.toluna.me (API Gateway)
  ├── OAuth Server (/oauth/*) — RFC 7591, PKCE, per-profile Cognito delegation
  └── Domain Proxy (/plm/{profile}/*) — routing, token swap, identity injection
         │ SigV4
         ▼
AgentCore Gateway (platform-tools-gw)
  ├── Interceptor — dynamic tool enrichment + profile filtering
  └── Targets → Handler Lambdas (one per MCP server)
```

## Components

| Component | Lambda | Purpose |
| --- | --- | --- |
| OAuth Server | mcp-oauth-server | Auth flow, token issuance, per-profile Cognito delegation |
| Domain Proxy | mcp-domain-proxy | Routing, token swap, tool name rewriting, identity injection |
| Interceptor | mcp-profile-filter | Dynamic tool enrichment + profile filtering |
| Datadog Handler | mcp-datadog-tools | Routes to datadog-mcp-server-py |
| Usain Bolt Handler | mcp-usain-bolt-tools | Routes to usain-bolt-mcp-server-py |
| ENV1 Handler | mcp-env1-tools | Routes to env1-mcp-server |

## Profiles

| Profile | Target | Handler |
| --- | --- | --- |
| `datadog` | datadog | mcp-datadog-tools |
| `usain-bolt` | usain-bolt | mcp-usain-bolt-tools |
| `env1` | env1 | mcp-env1-tools |
| `plm-kb` | confluence-platform-kb | Bedrock KB connector |
| `all` | — | All of the above |

## Auth Flows

**Default (datadog, usain-bolt):** User → our Cognito → Azure AD SSO → gateway JWT

**Per-profile (env1):** User → service's Cognito → Azure AD SSO → service token stored in DDB + gateway JWT

The handler Lambda reads the per-profile token from DynamoDB when tools need service-specific auth.

## Interceptor (Fully Generic)

Config via env vars only — no MCP-specific code:

- `HANDLER_LAMBDAS`: `{"datadog": "mcp-datadog-tools", "usain-bolt": "mcp-usain-bolt-tools", "env1": "mcp-env1-tools"}`
- `PROFILE_MAP`: `{"plm-kb": "confluence-platform-kb"}` (overrides only)
- `CACHE_TTL_SECONDS`: `300`

Flow: invokes each handler Lambda with `{"method": "tools/list"}` → caches → enriches gateway response → filters by profile.

## Adding a New MCP Server

1. Team publishes package to CodeArtifact
2. Platform creates handler Lambda + layer
3. Add to `HANDLER_LAMBDAS` + `GATEWAY_TARGETS` + `PROFILE_GATEWAYS`
4. If own Cognito: add to `PROFILE_COGNITO` on OAuth server + proxy
5. `terraform apply`
6. After initial setup: self-service via `./scripts/deploy-mcp.sh`

## Deploy Script

```shell
cd ~/code/git/toluna-main/remote-mcp-gateway/scripts
./deploy-mcp.sh <server-name> [source-path]
```

Handles full workflow: build layer → publish → update tfvars → terraform apply → verify.

## V2 Design — Dynamic MCP Server Targets

**Status:** Design approved, prototype planned. See [MCP Gateway — Generic Handler & V2](./mcp-gateway-v2-design.md).

Replace static `toolSchema` with **DYNAMIC MCP Server targets** using **Lambda Function URLs**: each handler speaks MCP protocol directly (Streamable HTTP); the gateway discovers tools by calling `tools/list` on every request; no static schema, no interceptor enrichment, no manual sync.

## Infrastructure

| Resource | Details |
| --- | --- |
| AWS Account | 457416912276 (datalab) |
| API Gateway | bg0u4yorok (custom domain: mcp.toluna.me) |
| AgentCore Gateway | platform-tools-gw-1h8ijbea8t |
| DynamoDB | mcp-oauth-clients, mcp-oauth-transactions |
| DNS (public) | Zone Z05854971UHYAY4ETIK6B |
| DNS (private) | Zone Z09663471W1AW33PSQC65 |

## Layer ARNs

| Package | Layer | Version |
| --- | --- | --- |
| datadog-mcp-server-py | datadog-mcp-server-layer | :13 |
| usain-bolt-mcp-server-py | usain-bolt-mcp-server-layer | :2 |
| env1-mcp-server | env1-mcp-server-layer | :1 |

## Terraform Structure

```
terraform/
├── main.tf              # Gateway, IAM, handler Lambdas, interceptor
├── variables.tf         # Configuration
├── outputs.tf           # Lambda ARNs
├── terraform.tfvars     # Layer ARNs, secrets
└── proxy/
    └── main.tf          # API Gateway, OAuth server, domain proxy
```
