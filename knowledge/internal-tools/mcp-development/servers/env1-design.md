---
title: ENV1 MCP Server — Design
description: Architecture and internal design of the ENV1 MCP server — one tool surface across two runtimes, auth as an injectable, and the v2 (decoupled) pending-change / apply flow it mirrors from env1.
tags: [plm, mcp, server, env1, design, v2-apply]
---

# ENV1 MCP Server — Design

**Purpose:** Architecture and internal design of the ENV1 MCP server. For the tool catalog and operational notes, see the parent [ENV1 MCP Server](./env1.md). For the general build/publish/deploy process, see the [MCP Development Guide](../docs/mcp-development-guide.md). Standards are in the [MCP Development Protocol](../steering/mcp-development-protocol.md).

## Design Goals

1. **One tool surface, two runtimes.** The same tool definitions and client run both locally (developer stdio, browser OAuth) and remotely (AgentCore gateway, DDB-backed token). Only the token provider differs.
2. **Auth as an injectable.** ENV1's genuine per-MCP need is the auth token, supplied as an injected `token_provider` rather than baked into the client.
3. **Advertised = executable.** Every tool in `TOOLS` must be dispatchable and backed by a real async client method. A contract test enforces this at build time.
4. **Mock parity.** The mock client implements the same method surface as the real client.
5. **Faithful to env1's own flows.** The MCP reuses env1's existing endpoints and mirrors the env1 UI (e.g. the v2 pending-change/apply flow). No env1 service/API changes are made.

## Package Layout

```
env1_mcp_server/
├── __init__.py           # __version__ (from package metadata), exports create_client
├── tool_definitions.py   # TOOLS list (plain dicts) + COMPAT_ROUTES
├── client.py             # Env1Client — async httpx wrapper over the ENV1 API
├── mock.py               # MockEnv1Client — canned responses, same method surface
├── factory.py            # create_client() — uniform construction entrypoint
├── context.py            # RequestContext dataclass — caller identity for v2 write/apply tools
├── server.py             # Local stdio MCP server (browser OAuth token provider)
└── auth.py               # LOCAL-ONLY: Cognito + Azure AD browser OAuth, local token cache
```

| File | Responsibility |
| --- | --- |
| `tool_definitions.py` | Source of truth for tool names, descriptions, schemas, and the `method` each maps to. No SDK imports. `additionalProperties: false` on all. |
| `client.py` | All ENV1 API calls. Registry GETs (no auth), S3 read/write (auth), repo/color lookups, and the v2 pending-change/apply flow. |
| `factory.py` | `create_client(token_provider=…)` — mock when `MOCK_MODE=true`, else the real client with the injected token provider. |
| `server.py` | Local runtime. Wires the browser token provider, lists tools, dispatches `call_tool`. Compact `json.dumps(result)`. |
| `auth.py` | **Local runtime only.** Browser OAuth, `localhost:3000/callback` handler, on-disk token cache. Never invoked in the remote gateway. |
| `context.py` | `RequestContext` carrying caller identity. Used by the v2 write/apply tools (`wants_context=True`). Read-only tools do not use it. |

## Tool Definition Contract

Each `TOOLS` entry carries a `method` field naming the async client method it dispatches to; may declare `"wants_context": True` to receive caller identity; every `inputSchema` sets `additionalProperties: false`. `COMPAT_ROUTES` maps deprecated names to current ones — ENV1 has none (`{}`).

## Request Flow

### Local (stdio) runtime

```
Agent (stdio)
  → server.py: call_tool(name, arguments)
      → dispatch by tool name → Env1Client.<method>(**arguments [, ctx])
          → [S3 tools] browser_token_provider() → auth.get_valid_token()/authenticate()
          → httpx call to https://env1.toluna.me/...
      → TextContent(compact JSON result)
```

### Remote (gateway) runtime

```
Agent → mcp.toluna.me/plm/env1
  → mcp-domain-proxy (Lambda, timeout 25s) → env1-handler (Lambda, timeout 30s, built from this package)
      → create_client(token_provider = DDB-backed provider)
          → token read from DynamoDB (session:{email}:env1)
          → [v2 tools] caller identity injected by the proxy → RequestContext
          → httpx call to https://env1.toluna.me/... (inside VPC)
      → JSON result
```

The dispatch logic and client are identical across both; only the **token provider** injected differs. `auth.py` participates in the local flow only. The **binding per-call timeout is the proxy's 25s**.

## Authentication Design

> **The auth in this package (`auth.py`) is LOCAL-ONLY.** The browser OAuth flow, the `localhost:3000/callback` handler, and the on-disk token cache (`~/.mcp-auth/env1-auth/token.json`) exist purely for the local stdio runtime. **The remote gateway uses its own authentication** — the Platform team's gateway build hook injects a DDB-backed token provider; the token is minted by the gateway's OAuth/Cognito flow and stored in DynamoDB. The two runtimes share only the `token_provider` injection point.

The S3 config tools require a token. The Terraform-registry tools and the service-catalog lookups (`get-service-by-repo`, `get-env-colors`) hit **public** env1 endpoints (mixing authed and public tools in one client is the documented env1 pattern). The v2 change/apply endpoints are also public — they take the caller identity in the request body, not a bearer token.

- **Token provider injection.** `Env1Client(token_provider=…)` accepts an async callable returning an auth token. Local: `browser_token_provider` → `auth.py`. Remote: a DDB-backed provider supplied by the gateway build hook.
- **401 retry.** `_authed_get` / `_authed_put` (S3 tools only) retry once after re-invoking the token provider on a 401, then raise `PermissionError`.
- **Cognito federation.** ENV1's own Cognito pool (`us-east-1_kB6XoEgW8`) federates to Azure AD; a pre-token Lambda injects a custom `role` claim.
- **Caller identity (for v2 write/apply).** The proxy injects the caller identity into the tool call; the dispatcher passes it through `RequestContext`. Independent of the auth token; **never** a tool-schema parameter.

## S3 Config Resolution

The ENV1 S3 layout is `{tribe}/{service}/{type}-env.json`, but callers only know the service name. The client resolves the rest: `GET /s3_storage/list?maxKeys=1000` (authenticated) → filter to files under `{service}/` with non-zero size → derive **tribe** from the bucket name (`toluna-{tribe}-tf-config-data`) → build `{tribe}/{service}/{type}-env.json`.

**Read:** bare service name → read every matching file (dict keyed by `app`/`data`/`shared`/…); `service/type` → just that one. **Write:** branches on legacy vs v2 (below).

## v2 (Decoupled) Write & Apply Flow

> **This is the central new behavior.** For a legacy service a config write is applied to S3 immediately. For a **v2 (decoupled) service**, env1 records a **pending change** that only takes effect once **applied** — the config is written to S3 *and* the PLM layer is deployed by the Shraga runner. The MCP mirrors the env1 UI exactly, using env1's existing endpoints. **No env1 service/API change is required.**

### Legacy vs v2 detection

env1's `service-config` DynamoDB record carries a `ui_version` field (`"legacy"` or `"v2"`). The MCP reads it via `GET /service/config/by_service?service=<name>` (or `by_repo`). `get-service-by-repo` is extended to surface `ui_version` (and `tribe`).

### Write behavior (`write-s3-json`)

env1's apply model is **env-centric**: `apply_change_to_json` merges a change into a single environment (`env_edit`) or replaces a workspace object in `shared-env.json` (`global_config`). So a v2 write must name the environment/workspace it targets — `write-s3-json` takes an `env` argument for v2 services (ignored for legacy).

- **legacy** → read-merge-PUT to S3 immediately → `{status: "applied"}`.
- **v2** (env REQUIRED) → do NOT PUT; map to env1's change model (`app`/`data` → `env_edit` with `{**data, envName: env}`; `shared` → `global_config` with `{env: data}`) → `POST /service/changes` (status `pending`, user `ctx.identity`) → `{status: "pending_apply", change_id, ...}`.

**Design choices:** env-scoped (matching env1's apply model — a whole-file blob would corrupt sibling environments); always-pending for v2 (the MCP does not replicate the UI's `requiresApply` rule engine; a redundant apply is at worst a no-op); no raw PUT for v2 (routes through `POST /service/changes` to preserve the pending record, service lock, and pre-apply snapshot).

### Apply behavior (`apply-changes`)

> **Critical: the apply is a TWO-part operation.** `POST /service/apply_changes` only orchestrates the Shraga/PLM runner — it does **not** write the S3 config file. The config write happens separately via `POST /service/config/apply_to_s3`. `apply-changes` must do **both** (write each change to S3 first, then trigger the runner). Omitting the S3 write was a bug — the apply would "succeed" while the config never landed.

The tool is **non-blocking**: it writes the changes to S3, triggers the runner, and returns a `batch_id`; the user checks status on demand with `get-apply-status`. It supports a `dry_run` preview. It resolves tribe, gathers pending changes, groups by workspace (`env_edit` → `prod*` → shared-prod else shared-non-prod; `global_config` → the workspace key), runs pre-checks (lock via `get_service_status.lockedBy`; open PRs via `check_open_prs`), then for each change: `apply_to_s3` (writes to S3, returns a snapshot) + `POST /service/changes/{id}/snapshot`, then `POST /service/apply_changes`. env1 applies **one workspace at a time** and auto-triggers the next.

### Checking completion (`get-apply-status`)

`GET /service/get_service_status?service=<name>` → `{service, status, lockedBy, change_statuses, done, error}`. `status` ∈ `active` | `applying` | `apply_failed` | `pending_apply`. `done` semantics are computed carefully so an idle service (returns `active` with no `change_statuses`) is not mistaken for a completed apply.

**On-demand check — do NOT auto-poll.** An apply runs Terraform via the Shraga runner and **typically takes 2–3 minutes**. `get-apply-status` is a fast single-shot check — call it **when the user asks**, and if still `applying`, report in-progress and check again only when the user next asks. Never poll on a timer.

**Why the tool doesn't block until done.** The `mcp-domain-proxy` Lambda timeout is **25 seconds** (the handler is 30s, but the proxy gives up first). A tool that sleeps until the apply finishes (2–3 min) would be killed at 25s and return nothing. So `apply-changes` returns immediately with a `batch_id`.

### End-to-end sequence (v2)

```
1. write-s3-json (env=qa)   → pending change created (change_id)         [status: pending_apply]
2. (repeat for more edits)
3. apply-changes (dry_run=true, optional) → preview the grouped plan     [status: dry_run]
4. apply-changes            → per change: apply_to_s3 (+ snapshot),      [status: applying, batch_id]
                              then apply_changes → SNS → runner applying
5. get-apply-status (only when the user asks — apply takes 2-3 min)
      ↳ applying   → still in progress; tell the user, check again later
      ↳ active     → all changes applied to S3 + PLM layer deployed       → done=true
        apply_failed → env1 reverts S3 from snapshot                      → done=true
6. (if multiple workspaces) env1 auto-triggers next workspace; check until all terminal
```

### env1 endpoints reused (all pre-existing, public)

Detect v2 vs legacy: `GET /service/config/by_service`; create pending change: `POST /service/changes`; list: `GET /service/changes?...&status=pending`; write change to S3 during apply: `POST /service/config/apply_to_s3`; store snapshot: `POST /service/changes/{id}/snapshot`; trigger runner: `POST /service/apply_changes`; check status: `GET /service/get_service_status`; open-PR pre-check: `GET /service/check_open_prs`.

## New / Changed Tools (summary)

| Tool | Change | Auth | Identity (`wants_context`) |
| --- | --- | --- | --- |
| `write-s3-json` | Branches on `ui_version`: legacy → immediate PUT; v2 → env-scoped pending change (`env` required). | S3: yes / changes: public | Yes |
| `get-service-by-repo` | Now also returns `ui_version`. | No (public) | No |
| `apply-changes` | **New.** Writes pending changes to S3 (`apply_to_s3` + snapshot) then triggers the runner; `dry_run` preview; returns `batch_id`; non-blocking. | Public | Yes |
| `get-apply-status` | **New.** On-demand completion check; returns `status` + `change_statuses` + `done`. | Public | No |

## Future Work — native async long-running operations

> **Known limitation.** The apply is genuinely long-running (2–3 min), but the MCP transport as deployed has no server-push / progress mechanism. Completion is checked on demand via `get-apply-status`.

The gateway timeouts (verified in `remote-mcp-gateway` Terraform): `mcp-domain-proxy` 25s (binding), `mcp-env1-tools` handler 30s, API Gateway ~30s. **Decision (Aug 2026): do NOT add a bounded `wait-for-apply` tool** — at a 25s ceiling a safe internal wait covers under ~15% of a 2–3 min apply. The real fix (progress notifications / an operation handle the client awaits natively) is a **platform/gateway capability** tracked against the [MCP Gateway V2 work](../docs/mcp-gateway-v2-design.md).

## Build-Time Contract Test

`tests/test_contract.py` asserts: required keys + unique names; every tool's `method` resolves to an async client method; every input-schema property is accepted by the method; `wants_context` methods take `ctx`; compat routes target real, non-advertised tools. Run: `MOCK_MODE=true pytest tests/test_contract.py`.

## Testing Strategy

`MockEnv1Client` mirrors the real client's method surface with canned responses including the v2 flow (returns `ui_version="v2"` for a designated service, records env-scoped pending changes, simulates the S3-write step, supports `dry_run`, steps a mock apply `applying` → `active`). `factory.create_client()` honors `MOCK_MODE=true`. `tests/test_tools.py` validates tool-definition standards + mock behavior. Suite: 33 tests.

## Related

- Parent: [ENV1 MCP Server](./env1.md) (tool catalog, operational notes)
- [MCP Development Guide](../docs/mcp-development-guide.md)
- [MCP Development Protocol](../steering/mcp-development-protocol.md)
- [MCP Gateway — Generic Handler & V2](../docs/mcp-gateway-v2-design.md)
