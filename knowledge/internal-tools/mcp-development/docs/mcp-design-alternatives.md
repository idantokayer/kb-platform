---
title: MCP Design Alternatives & Rejected Options
description: A living ADR log of MCP architecture options decided against (e.g. FastMCP) and the production incidents that motivated each standing rule, so decisions aren't re-litigated.
tags: [plm, mcp, design, adr, incidents]
---

# MCP Design Alternatives & Rejected Options

**Status:** Living document · **Purpose:** Record architectural options we evaluated and **decided against**, with the reasoning, so the same question doesn't get re-litigated from scratch.

Part of [MCP Development](../index.md). Each entry is a lightweight ADR.

---

## ADR-001 — FastMCP as our core framework

**Date:** 2026-08-31 · **Status:** Rejected (for existing servers and for V2).

**Context:** [FastMCP](https://gofastmcp.com) is a mature Python framework for building MCP servers (it seeded the high-level API in the official MCP Python SDK). Should we adopt it as the base for our MCP servers?

### What FastMCP offers

Schema generation from type hints + docstrings (`@mcp.tool`); input/output validation; transports (stdio + Streamable HTTP); generic OAuth auth; client libraries; "Apps"; and **Horizon** — their *commercial* enterprise MCP gateway.

### Why it is NOT a good fit for us

1. **We already use the industry-standard SDK it sits on top of.** All four servers depend on the official `mcp` SDK; FastMCP is a convenience layer over the same SDK.
2. **Its central feature conflicts with our load-bearing design choice.** FastMCP makes the live server object the source of truth; our source of truth is deliberately the opposite — `tool_definitions.py` (plain, SDK-free data). That decoupling is what enables two runtimes from one pure package and the generic handler + contract test.
3. **Its auth model doesn't map to ours.** We use per-profile Cognito delegation, a DDB-backed token provider, claim-bearing JWTs, and `wants_context` identity. FastMCP's auth is generic OAuth.
4. **It solves a problem we don't have.** The friction it removes is "writing `inputSchema` dicts by hand" — not our bottleneck.
5. **Even for V2, it doesn't earn its complexity.** V2 is almost entirely a transport wrapper over machinery we already have.
6. **Strategic tell.** FastMCP's monetization is Horizon, an enterprise MCP gateway — a direct analog to our `remote-mcp-gateway`.

### Decision

**Do not adopt FastMCP** — not wholesale, not as an extended base, not for V2 core. Continue using the official `mcp` Python SDK directly.

### Conditions that would make us revisit

A greenfield, standalone MCP server with no gateway, single runtime, and generic OAuth; or the official `mcp` SDK stops being maintained; or schema-authoring burden grows enough that annotation-derived schemas become a measurable win.

---

## Incidents & Derived Rules

Production incidents that motivated a standing rule. The **rule** lives in the [Protocol](../steering/mcp-development-protocol.md) / [Guide](./mcp-development-guide.md); the **story** lives here.

### INC-001 — Dispatch drift (usain-bolt: advertised ≠ executable)

**What happened:** usain-bolt's `tool_definitions` advertised 16 tools, but the handler's hand-written dispatch chain routed only 7 — the other 9 returned `Unknown tool`.
**Derived rule:** the generic handler dispatches from the single-source `TOOLS` table via each entry's `method` binding, and `tests/test_contract.py` asserts every advertised tool resolves to a real async method — making "advertised but not executable" a build failure.

### INC-002 — Silent injectable-drop (chamberlain: remote auth dead no-op)

**What happened:** chamberlain's `create_client(**_ignored)` swallowed the injected `token_provider`. The client fell back to a local env-var default absent in Lambda — remote auth broken while all tests were green.
**Derived rule:** `create_client` MUST name and thread each injectable; a Pattern-A package's contract test MUST assert `create_client(token_provider=sentinel)` yields a client whose provider *is* that sentinel.

### INC-003 — Stale-layer deploy (env1: bumped-but-not-published)

**What happened:** an env1 version was bumped in git and merged, but never published to CodeArtifact. The deploy shipped the *previous* published version — prod silently ran stale code.
**Derived rule:** `deploy-mcp.sh` runs a **publish-freshness guard** before building — hard-fails if the repo is ahead of CodeArtifact. Publish → then deploy; merge to main → then publish.

### INC-004 — Broken-mock propagation (normalizer PR: red main spreads)

**What happened:** a normalizer PR merged to `main` with broken pulse-status test mocks. With no green-suite gate, the breakage propagated to every branch that later merged main.
**Derived rule:** a PR must have a green suite before review; if `main` is red, fix it first; cross-feature merges require a full test pass; mocks must exercise the real code path.

### INC-005 — Auth-model confusion (chamberlain: "we have Cognito" ≠ remote path wired)

**What happened:** "chamberlain has its own Cognito" was taken to mean the remote auth path was ready. It wasn't — a Pattern-A service needs four pieces split across the service team and the platform.
**Derived rule:** the Pattern-A remote path is a **4-step checklist split by actor** — service-side (callback URL + auth-code flow) vs. gateway-side (OAuth-server + handler `PROFILE_COGNITO` + DDB IAM + delegation).

### INC-006 — Identity on the AgentCore gateway (V1 limitation → the `_user_email` workaround)

**What happened:** while trying to remove what looked like a redundant middle layer — the proxy injecting the caller's email into tool arguments (`_user_email`) — a redacted handler probe (2026-09-01) proved **AgentCore direct-invoke targets deliver to the handler ONLY the tool arguments plus AgentCore's own plumbing keys** — no user identity, no forwarded JWT. So on V1 the caller's email (needed by Pattern-A hooks as the DDB session-lookup key) has no out-of-band channel to the handler. Removing the injection would have broken env1 + chamberlain remote auth.
**Derived rule:** the proxy decodes the caller's email from the gateway JWT and injects it into tool arguments (`_user_email` for env1/chamberlain; `username` into usain-bolt's real `owners`). **This is a workaround for a gateway limitation, NOT the intended design — do NOT "clean it up" on V1.** The architecturally-correct state (identity strictly out-of-band via `ctx.identity`) is delivered by **V2** (Function-URL targets receive the real request incl. the `Authorization` header). Distinction: `token_provider` (Pattern A) is *auth*; `ctx.identity` (Pattern B) is *identity-as-a-value*. See [MCP Gateway V2 → Identity](./mcp-gateway-v2-design.md).

---

## Template for new entries

```
## ADR-NNN — <option / question>

**Date:**
**Status:** Rejected / Accepted / Superseded by ADR-XXX
**Context:** <what prompted the question; state of the system at the time>

### What it offers
### Why it does / doesn't fit us
### Decision
### Conditions that would make us revisit
```
