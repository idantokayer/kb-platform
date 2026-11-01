---
title: MCP Development Guide
description: The full spec and procedure for building, testing, publishing, and shipping a Toluna MCP server — package structure, the tool_definitions pattern, contract test, auth patterns, and rollout.
tags: [plm, mcp, development, guide, how-to]
---

# MCP Development Guide

**Purpose:** The full spec + procedure for building, testing, publishing, and shipping a Toluna MCP server. This is the **how-to**; the [MCP Development Protocol](../steering/mcp-development-protocol.md) is the terse **rules**.

**Audience:** a developer who owns a Toluna service and wants to expose it for agentic access as a remote MCP under `mcp.toluna.me`.

Part of [MCP Development](../index.md).

---

## How the docs fit together (read this first)

| Doc | What it is | Where |
| --- | --- | --- |
| [MCP Development Protocol](../steering/mcp-development-protocol.md) | The rules you always follow while building. Terse, normative. | Steering (always in your agent's context) |
| **MCP Development Guide** (this doc) | The full spec + step-by-step procedure. Your primary how-to. | Confluence (reference) |
| [MCP Platform Operations](../steering/mcp-platform-operations.md) | The gateway operator's runbook. The Platform team's, not yours. | Steering (Platform project) |

### Installing the steering + guide into your agent (Kiro example)

The Protocol is a **steering file** — a markdown file your coding agent loads into context automatically.

1. **Get the steering file locally.** Copy the MCP Development Protocol into `.kiro/steering/mcp-development-protocol.md` (Kiro auto-loads every `.md` under `.kiro/steering/`).
2. **Custom agents must opt in** — add to config: `{ "resources": ["file://.kiro/steering/**/*.md"] }`
3. **This Guide stays in Confluence** — reference material you fetch on demand (e.g. via the `plm-kb` MCP), not always-on context.

> **Rule of thumb:** rules → steering (always on). Spec/procedure → this Guide (on demand). Operator runbook → Platform's project only.

---

## Getting Started — New MCP Server from Scratch

You build and publish the **package** (Steps 1–7). The Platform team owns the gateway and deploys it (Steps 8–9).

### Step 1: Create the repository

Name format: `{service-name}-mcp-server-py`, under `tolunaengineering` in Bitbucket.

### Step 2: Scaffold the package

```
{service-name}-mcp-server-py/
├── src/{service_name}_mcp_server/
│   ├── __init__.py            # derives __version__ from package metadata (NEVER hardcoded)
│   ├── tool_definitions.py    # ← START HERE — TOOLS (def + method binding) + COMPAT_ROUTES
│   ├── context.py             # RequestContext dataclass (caller identity)
│   ├── client.py              # async httpx client; every tool method is async
│   ├── factory.py             # create_client(**injectables) honoring MOCK_MODE
│   ├── server.py              # local stdio MCP server (imports from tool_definitions)
│   └── mock.py                # mock client — mirrors the real client's method surface
├── tests/
│   ├── test_contract.py       # HARD GATE — advertised == executable
│   ├── test_tools.py
│   └── conftest.py
├── pyproject.toml
└── README.md
```

`pyproject.toml` essentials:

```toml
[build-system]
requires = ["setuptools>=61"]
build-backend = "setuptools.build_meta"

[project]
name = "{service-name}-mcp-server-py"
version = "0.1.0"                       # THE single source of truth for the version
requires-python = ">=3.11"
dependencies = ["httpx>=0.24", "mcp>=1.1.2"]   # pin exact versions in practice

[tool.setuptools.packages.find]
where = ["src"]                         # src/ layout
```

### Step 3: Define your tools — `tool_definitions.py`

The **single source of truth**. Plain dicts, no MCP SDK import. Each entry binds a tool's schema to the client method that executes it:

```python
TOOLS = [
    {
        "name": "get-something",                # hyphenated verb-noun; advertised to agents
        "method": "get_something",              # REQUIRED: the async client method it dispatches to
        "description": "What this does. Include parameter examples and defaults.",
        "inputSchema": {
            "type": "object",
            "properties": {
                "param1": {"type": "string", "description": "Example: 'prod', 'qa-b'"},
            },
            "required": ["param1"],
            "additionalProperties": False,      # ALWAYS — stops the LLM inventing params
        },
        # "wants_context": True,                # OPTIONAL — receive caller identity (see Auth)
    },
]

# Deprecated tool names → current names. Not advertised; resolved if called directly.
COMPAT_ROUTES = {}
```

**Rules:** hyphenated verb-noun names; plain dicts only; `additionalProperties: False` on every schema; **schema property names must match the client method's parameter names**. Must include a `get-version` tool.

### Step 4: Implement `client.py`

Thin async `httpx` wrapper. **Every tool method is** `async def`. Raise/return structured errors; reasonable timeouts (30s default).

### Step 5: The `create_client` factory — `factory.py`

The **one construction entrypoint**, used by both `server.py` (local) and the gateway handler (remote). It honors `MOCK_MODE` and threads any injectables through to the client.

> **Critical:** if your service needs auth, the factory MUST *name and forward* `token_provider`. A factory that only has `(**kwargs)` silently drops `token_provider`, the client falls back to a local default that doesn't exist in Lambda, and **remote auth fails silently while every test stays green.** The contract test `test_token_provider_is_threaded` enforces this.

### Step 6: `server.py` — local stdio server

Registers tools generically from `TOOLS` (a loop, not N hand-written branches), awaits async methods, serializes compact JSON. Dispatches by `method`, passes `ctx` only to `wants_context` tools — mirroring the generic handler exactly, so local and remote behave identically.

### Step 7: Test, then publish

```shell
pip install -e .
MOCK_MODE=true python3 -m pytest tests/        # contract test + unit tests must be green
python3 -m build --no-isolation                # --no-isolation: our pip index lacks build backends
aws codeartifact login --tool twine --repository platform-pypi --domain toluna-artifact-repository --domain-owner 014931512072 --region us-east-1
python3 -m twine upload --repository codeartifact dist/*
```

### Step 8: Request gateway deployment (Platform team)

You cannot deploy to the gateway yourself. Contact the Platform team with: package name + version on CodeArtifact; desired profile name; whether the handler needs VPC access; whether your service has its own Cognito pool + custom claims (→ auth Pattern A).

### Step 9: Subsequent versions

Bump `pyproject.toml` version → build → publish → notify Platform. **Publish before deploy** — the layer is built from CodeArtifact, not your working tree.

### Step 10: Document

Add a `Servers/` page for your MCP; update the main MCP Development server table.

---

## The Generic Handler Contract (why the package looks like this)

The gateway runs **one generic handler** for every MCP. It dispatches from your `TOOLS` table via each entry's `method` binding — there is no hand-written per-server dispatch. This makes "advertised but not executable" structurally impossible.

Consequences: `method` is explicit in every TOOLS entry; args derive from `inputSchema.properties` (schema names must equal method param names); all client methods are async; the mock mirrors the real client's method surface; the package is pure (no gateway/platform code). See [MCP Gateway — Generic Handler & V2](./mcp-gateway-v2-design.md).

### Build-time contract test (HARD GATE)

Ship `tests/test_contract.py` — it makes "advertised but not executable" a **build failure**. It asserts: every tool's `method` exists on the client and is async; args resolve from schema; `wants_context` methods take `ctx`; compat routes target real tools. For a Pattern-A (authed) package, also assert the injection point is live (`test_token_provider_is_threaded`) — the guard that catches the silent factory-swallow bug.

---

## The `get-version` tool (required)

Every MCP must expose it — agents use it to confirm which version is deployed. Method `get_version` returns `{"name", "version", "description"}`, with the version derived from package metadata.

## Version Management — Single Source of Truth

**There is exactly ONE place a human types a version:** `pyproject.toml`. `__init__.py` derives it at import via `importlib.metadata.version(...)`. Never hardcode `__version__`. To release: edit `pyproject.toml` version — the only line you touch.

---

## Per-Service Auth — dev-facing

Your tool schema **never** contains an auth token or caller identity. Those arrive out-of-band from the runtime.

| Your service… | Pattern | Reference |
| --- | --- | --- |
| authorizes on its **own Cognito + custom claims** | **A — token provider** | env1, chamberlain |
| just needs the **caller's identity** (attribution) | **B — RequestContext** | usain-bolt |
| pure read, no identity | neither | datadog |

Two different things flow via two channels. **`token_provider`** (Pattern A) is *authentication* — an async callable yielding the service's own bearer token, injected into the client via `create_client`; tools never see it. **`ctx.identity`** (Pattern B) is *identity as a value* — the caller's email, passed per-call as `ctx` only to tools declaring `wants_context: True`.

### Pattern A — token provider

The client takes an injected `token_provider` (async callable returning a bearer token). The provider differs by runtime (local env var vs. remote DDB); the client code doesn't. A 401 refreshes via the SAME provider — never a hardcoded browser flow. Your whole job: (a) accept `token_provider` in the client `__init__`, (b) thread it through `create_client`, (c) never put a token/identity in the schema. The gateway's `build_client` hook is Platform's.

**Service-team prerequisites for the REMOTE path:** on your own Cognito pool, (1) add the MCP OAuth callback `https://mcp.toluna.me/oauth/cognito/callback` to allowed callback URLs, and (2) enable the `authorization_code` flow on the app client.

### Pattern B — RequestContext (identity only)

Declare `wants_context: True` on the tool; its client method takes `ctx` with a safe default. No token provider, no Cognito, no hook. Auth/identity is per-tool, not per-flow — a tool gets identity only if it actually needs it.

---

## Testing

Unit tests in mock mode (`MOCK_MODE=true pytest`); the contract test is a hard gate; mocks must exercise the real code path AND mirror the real client surface; new tools require new tests; a PR must have a green suite before review; cross-feature merges need a full test pass.

## Phased Rollout

1. **Read-only tools** — ship immediately.
2. **Filtered/computed tools** — derived from read data.
3. **Write tools** — require auth + a dry-run/confirmation path + explicit scoping (never an "all" default).

### Write-tool safety mechanics

A write tool needs all three, verified in review: **auth** (Pattern A or B); **dry-run / confirmation** (a `dry_run` flag returning the diff, or a two-phase `preview-x` → `execute-x` with an expiring `plan_id`); **explicit scoping** (a required scope arg for destructive/all-target writes — a missing scope is an error, not implicit "all"). **Long-running writes: return a handle, don't block** — the gateway proxy has a ~25s per-call ceiling, so kick off the work, return a `batch_id`/`plan_id`, and expose a separate status/read tool. Reference: env1's `apply-changes` + `get-apply-status`; chamberlain's `preview-deploy-sync` → `execute-deploy-sync`.

## Token Efficiency

LLMs pay per token. `json.dumps(result)` — never `indent=2`. Strip fields the LLM doesn't need; return summaries + only relevant items; for comparisons, return differences only.

## Error Handling

Return structured errors the LLM can act on: `{"error": "...", "details": "..."}`. Never expose raw stack traces, internal URLs, or infra details.

---

## Related

- [MCP Development Protocol](../steering/mcp-development-protocol.md) — the rules (steering)
- [MCP Gateway — Generic Handler & V2](./mcp-gateway-v2-design.md) — architecture
- [MCP Design Alternatives & Rejected Options](./mcp-design-alternatives.md) — rejected options + incidents
- Servers/ — one page per existing MCP for reference
