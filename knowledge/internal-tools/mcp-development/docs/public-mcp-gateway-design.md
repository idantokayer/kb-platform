---
title: Public MCP Gateway — Design
description: Design for a public-facing MCP server for Toluna's external clients, authenticated with their existing TolunaExternalAPI key via a separate isolated AgentCore gateway.
tags: [plm, mcp, gateway, design, public, external-api]
---

# Public MCP Gateway — Design Document

**Status:** Draft · **Date:** August 11, 2026 · **Author:** Raanan Gavrielli · **Stakeholders:** Alon (MCP developer), Platform team (gateway owners)

Part of [MCP Development](../index.md). Contrast with the internal [Remote MCP Gateway](./remote-mcp-gateway.md).

## 1. Purpose

Expose a public-facing MCP server for Toluna's external clients (P&G, Reckitt, IFF, etc.) — the same customers who already use the TolunaExternalAPI. Clients authenticate with their existing API key. No new credentials needed.

## 2. Context

**Existing commercial API:** Account `415875511270` (ts-api-prod); API Gateway `ts-api-gw-prod-green` (regional REST API); auth via `x-api-key` header; usage plan 300 RPS burst, 500k calls/month; ~27 API keys tagged with `TEF_ID`; the API internally resolves API key → TEF_ID → scoped data access.

**Internal MCP platform:** `mcp.toluna.me` — Azure AD SSO, internal tools only; AgentCore Gateway pattern; NOT to be exposed publicly.

## 3. Architecture

```
External Client (AI Agent)
  Config: mcp-remote, args: ["https://api.toluna.me/mcp", "--header", "x-api-key:CLIENT_KEY"]
  │ HTTPS + x-api-key
  ▼
api.toluna.me/mcp (API Gateway in ts-api-prod, 415875511270)
  • Same API GW as TolunaExternalAPI (new /mcp routes)
  • API key validation via existing usage plan; rate limiting; key passes through to Lambda
  │ Lambda invocation (key in context)
  ▼
MCP Public Proxy Lambda (new, in ts-api-prod)
  • Extracts API key; forwards MCP request to AgentCore Gateway (SigV4 cross-account)
  • Passes API key in custom context; handles MCP protocol; no identity resolution
  │ SigV4 (cross-account) + API key in context
  ▼
AgentCore Gateway (new: public-mcp-gw, in datalab 457416912276)
  • Separate gateway — isolated from internal tools; IAM auth; interceptor; single target: Alon's MCP
  │ Lambda invocation
  ▼
Handler Lambda (in datalab, VPC if needed)
  • Alon's MCP package as Lambda layer; responds to tools/list; routes tools/call; passes API key downstream
  │ HTTPS + x-api-key (pass-through)
  ▼
TolunaExternalAPI (existing, ts-api-prod)
  • Resolves API key → TEF_ID → client identity; enforces permissions; scopes data
```

## 4. Key Design Decisions

- **4.1 Same API key for MCP and downstream** — the client's existing API key both authenticates the MCP request (API Gateway validates it) and authenticates downstream API calls (handler passes it to TolunaExternalAPI). No new credentials, no token exchange, no identity mapping.
- **4.2 Separate AgentCore Gateway** — public MCP tools run on a dedicated `public-mcp-gw`, isolated from internal `platform-tools-gw`. Prevents accidental exposure of internal tools, cross-contamination of auth models, and blast radius.
- **4.3 MCP server developed by Alon's team, deployed by Platform team** — follows the standard protocol. The handler is generic (owned by Platform).
- **4.4 API key pass-through (no identity resolution in proxy)** — the proxy passes the API key to the handler, which passes it to the downstream API. The TolunaExternalAPI already handles all authorization.

## 5. Client Configuration

```json
{
  "toluna": {
    "command": "mcp-remote",
    "args": [
      "https://api.toluna.me/mcp",
      "--header",
      "x-api-key:CLIENT_API_KEY"
    ],
    "timeout": 120000
  }
}
```

No OAuth, no browser flow, no SSO. The `--header` format is `name:value` (no space after colon). Works with any MCP client that supports `mcp-remote` (Kiro, Claude, Cursor, etc.).

## 6. MCP Protocol Considerations

- **6.1 Auth model** — `mcp-remote` supports static custom headers via `--header`; no OAuth flow.
- **6.2 Streamable HTTP** — the public gateway should speak MCP Streamable HTTP directly (not SSE).

## 7. Infrastructure

**New resources (ts-api-prod, 415875511270):** API Gateway routes `/mcp`, `/mcp/{proxy+}` on `ts-api-gw-prod-green`; Lambda `mcp-public-proxy`; cross-account IAM role.

**New resources (datalab, 457416912276):** AgentCore Gateway `public-mcp-gw` (IAM auth); handler Lambda `mcp-public-<service-name>-tools`; Lambda layer from Alon's package; IAM roles.

**DNS:** `api.toluna.me` (likely already exists).

## 8. Security

| Concern | Mitigation |
| --- | --- |
| Rate limiting | Existing usage plan quotas apply (300 burst, 500k/month) |
| API key exposure | Same risk as existing API — no change |
| Internal tool exposure | Separate gateway, no shared targets |
| Data scoping | Handled by TolunaExternalAPI (TEF_ID resolution) |
| Cross-account access | IAM role with minimal permissions (invoke gateway only) |

## 9. Implementation Plan

- **Phase 1: Foundation** — create AgentCore Gateway in datalab; proxy Lambda in ts-api-prod; cross-account IAM role; `/mcp` routes; verify `api.toluna.me` routing.
- **Phase 2: Alon's MCP Server** — build + publish package; Platform deploys handler + layer + target; end-to-end test with a test API key.
- **Phase 3: Client Rollout** — client documentation; announce; monitor via API Gateway metrics.

## 10. Open Questions

| # | Question | Owner | Status |
| --- | --- | --- | --- |
| 1 | Does `mcp-remote` support static custom headers? | Platform | ✅ Yes — `--header "x-api-key:VALUE"` |
| 2 | What tools is Alon planning? | Alon | Open |
| 3 | Does `api.toluna.me` already exist as a domain? | Platform | Open |
| 4 | Own usage plan or share `prodPlan`? | Platform + Alon | Open |
| 5 | Cross-account networking — can Lambda in ts-api-prod call AgentCore in datalab? | Platform | Open |

## 11. Comparison: Internal vs Public MCP

| Aspect | Internal (`mcp.toluna.me`) | Public (`api.toluna.me/mcp`) |
| --- | --- | --- |
| Auth | Azure AD SSO (OAuth2 + PKCE) | API key (x-api-key header) |
| Users | Toluna employees | External clients |
| Gateway | `platform-tools-gw` + `confluence-kb-gw` | `public-mcp-gw` (new) |
| Tools | 6 profiles, 30+ tools | Single MCP server (Alon's) |
| Account | datalab (457416912276) | Proxy: ts-api-prod, Gateway: datalab |
| Rate limits | None (internal) | Usage plan (300 RPS, 500k/month) |
| Identity | Email from JWT | TEF_ID from API key (resolved by downstream API) |
