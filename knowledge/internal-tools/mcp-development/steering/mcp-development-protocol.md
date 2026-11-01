---
title: MCP Development Protocol
description: The terse, normative rules for building a Toluna MCP server — naming, required files, the single-source tool contract, out-of-band auth, build-time gates, release order, and write-tool safety.
tags: [plm, mcp, steering, protocol, rules]
---

# MCP Development Protocol

**Status:** Active — the **rules**. For the full how-to, see the [MCP Development Guide](../docs/mcp-development-guide.md).

Part of [MCP Development](../index.md) (Steering).

## Who You Are

You own a **Toluna service** (a deployment tool, a config API, an observability surface, a test runner — something with an HTTP API) and you want to **expose it for agentic access**. You do that by building a **remote MCP server** hosted under Toluna's shared MCP gateway (`mcp.toluna.me`). You are a **consumer of the platform, not its operator** — the Platform team owns the gateway, the deploy pipeline, and the shared infrastructure. Your job is to build a correct MCP **package** that follows these standards, then hand it to the platform to deploy.

**How you build here:** follow the rules in this document (each prevented a real production bug); the [MCP Development Guide](../docs/mcp-development-guide.md) is your how-to; your MCP is a thin client (authorization, business rules, and audit stay in your service's API); advertised must equal executable; ship read-only first, write tools with care; when in doubt, match the reference implementations (env1, usain-bolt).

## Naming Convention

- Repository / package: `{service-name}-mcp-server-py`
- Tool names: hyphenated verb-noun (`get-versions`, `search-logs`, `execute-run`)
- Gateway profile: lowercase, hyphenated (`usain-bolt`, `plm-kb`)

## Required Files

```
src/{service_name}_mcp_server/
├── __init__.py            # derives __version__ from package metadata (never hardcoded)
├── tool_definitions.py    # SINGLE SOURCE OF TRUTH — TOOLS (def + method) + COMPAT_ROUTES
├── context.py             # RequestContext (package-owned shape)
├── client.py              # async httpx client; all tool methods async
├── factory.py             # create_client(**injectables) honoring MOCK_MODE
├── server.py              # local stdio MCP server (imports from tool_definitions)
└── mock.py                # mock client — mirrors the real client surface
tests/test_contract.py     # HARD GATE — advertised == executable
```

Full scaffold + code: [Guide → Getting Started](../docs/mcp-development-guide.md).

## Code Standards

- Python 3.11+; async by default (httpx + asyncio)
- Pinned dependencies (exact versions) in `pyproject.toml`
- No stdout pollution in the server (stderr/logging)
- Validate inputs; return structured error dicts, never raw exceptions/stack traces
- `additionalProperties: false` on every tool `inputSchema`

## Tool Design

One tool = one clear purpose; hyphenated verb-noun name; required params only when truly necessary; return compact JSON (`json.dumps(result)` — **never** `indent=2`); strip fields the LLM doesn't need; `additionalProperties: false` to stop the LLM inventing params.

## The Tool Contract (single source → generic dispatch)

The gateway runs **one generic handler** for every MCP; it dispatches from your `TOOLS` table. There is no hand-written per-server dispatch. Therefore:

- `tool_definitions.py` is the single source of truth — plain dicts, no SDK import. Each entry carries `name`, `description`, `inputSchema`, and `method`.
- **Schema property names MUST match the client method's parameters.** If they diverge, **fix the schema to match the method — never add an arg-mapping shim**.
- **All client methods are** `async` — including `get_version`.
- `COMPAT_ROUTES` (deprecated→current names) lives in `tool_definitions.py`.
- **The package is pure** — no gateway/platform code. It runs identically local (stdio `server.py`) and remote (generic-handler Lambda); only injectables differ.

See [MCP Gateway — Generic Handler & V2](../docs/mcp-gateway-v2-design.md).

## Identity & cross-cutting fields (out-of-band, never in the schema)

Caller identity and auth tokens are **never** tool-schema parameters — the LLM must never see or fill them. They arrive out-of-band via a package-owned `RequestContext` (identity) or an injected token provider (auth), passed only to tools that declare `wants_context: True`. Full mechanics: [Guide → Auth](../docs/mcp-development-guide.md).

## `create_client` factory — thread every injectable

`create_client(**injectables)` (in `factory.py`) is the one construction entrypoint, honoring `MOCK_MODE`. **It MUST name and forward each injectable** (e.g. `token_provider`) to the client — a bare `**kwargs` that swallows it makes remote auth a silent no-op while every test stays green. Reference: env1's `factory.py`.

## Required Tool: `get-version`

Every MCP MUST expose `get-version` (empty schema, `additionalProperties: false`), backed by an async `get_version()` returning name/version/description from the dynamic `__version__`.

## Version — Single Source of Truth

**Exactly ONE place a human types a version:** `pyproject.toml`. `__init__.py` derives it at import via `importlib.metadata` — **never hardcode** `__version__`.

## Per-Service Auth — pick the lightest pattern

| Your service… | Pattern | Reference |
| --- | --- | --- |
| authorizes on its **own Cognito + custom claims** | **A — token provider** | env1, chamberlain |
| just needs the **caller's identity** (attribution) | **B — RequestContext** (`wants_context`) | usain-bolt |
| pure read, no identity | neither | datadog |

**Universal rule:** tokens/identity are never in `inputSchema` — always out-of-band.

- **Pattern A:** client takes an injected `token_provider`; never hardcodes a browser flow; 401 → call the provider again. A Pattern-A service can still have public endpoints (mixing is normal). Remote-path prerequisites are a **4-step checklist split by actor** — *you* (service team) add the MCP OAuth callback `https://mcp.toluna.me/oauth/cognito/callback` to your app client's allowed callback URLs and enable the `authorization_code` flow; the Platform team wires the gateway side.
- **Pattern B:** declare `wants_context: True`; the method reads `ctx.identity` (default `"local-dev"`). No token, no Cognito, no hook.

## Build-Time Gates (hard — a PR must pass all)

1. **Contract test** (`tests/test_contract.py`) — every `TOOLS.method` exists on the client & is async; `wants_context` ⇔ method takes `ctx`; args resolve from schema; `COMPAT_ROUTES` target real, non-advertised tools. Pattern-A: also assert `create_client(token_provider=sentinel)` threads it.
2. **Green suite before review** — `MOCK_MODE=true pytest` all green. Never merge on a red suite.
3. **Version single-source** — version only in `pyproject.toml`.
4. **Flatten, don't hook** — schema names match method params; no arg-mapping shim.

Rationale (the incidents that produced them): [Design Alternatives → Incidents & Derived Rules](../docs/mcp-design-alternatives.md).

## Order of Operations (release)

**merge → publish → deploy.** In that order, always. Merge to main before you publish (`main` is the source of truth); publish to CodeArtifact before you deploy (the Lambda layer is built from CodeArtifact, not your working tree — `deploy-mcp.sh` has a publish-freshness guard that hard-fails if the repo is ahead).

## Phased Rollout (convention, not a gate)

1. Read-only tools — ship immediately.
2. Filtered/computed tools — derived from read data.
3. Write tools — see Write-Tool Safety below.

## Write-Tool Safety

A write tool MUST have **all three** — enforce in review, since no build-time gate catches their absence:

- **Auth** — Pattern A/B as applicable; the caller's identity/token flows to the service, which authorizes. (Identity is per-tool: the mutating tools take it, a companion read/status tool need not.)
- **Dry-run / confirmation path** — the tool can preview *exactly* what would change without doing it (a `dry_run` flag, or a two-phase preview→execute with an expiring plan id).
- **Explicit scoping** — no **unbounded** blast radius. A missing scope on a destructive/all-target write is an error, never an implicit "all." This does **not** forbid a safe, ordered, previewable default *sequence* (e.g. env1 applies workspace-by-workspace from a `service`-only call).
- **Long-running → return a handle, don't block.** The gateway proxy has a ~25s per-call ceiling, so a longer write must be non-blocking: return a handle (`batch_id`/`plan_id`) + status and expose a separate on-demand status tool.

Reference: chamberlain's `preview-deploy-sync` → `execute-deploy-sync`; env1's v2 `apply-changes` + `get-apply-status`.

## Security

- Never commit credentials; `.env.example` for templates; secrets in Secrets Manager / SSM
- OAuth tokens cached in `~/.mcp-auth/` locally, DynamoDB remotely
- Write tools: auth + dry-run + explicit scoping

## Token Efficiency

`json.dumps(result)` (never `indent=2`); strip fields the LLM doesn't need; summaries + relevant items only; comparisons return differences only.

## Error Handling

Return structured errors the LLM can act on: `{"error": "...", "details": "..."}`. Never expose raw stack traces, internal URLs, or infra details.

## Where things live

- **This doc (steering)** — the rules, always in context.
- [MCP Development Guide](../docs/mcp-development-guide.md) — full spec, procedure, worked code, auth walkthroughs.
- [MCP Platform Operations](./mcp-platform-operations.md) — gateway operator runbook. Platform team.
- [MCP Gateway — Generic Handler & V2](../docs/mcp-gateway-v2-design.md) — architecture.
- [Design Alternatives & Incidents](../docs/mcp-design-alternatives.md) — rejected options + incidents behind these rules.
