---
title: MCP Gateway — Generic Handler & V2 (Design)
description: Design for one generic gateway handler dispatching from each package's TOOLS table (active), plus a demand-driven V2 transport swap to dynamic MCP-server targets over Lambda Function URLs.
tags: [plm, mcp, gateway, design, generic-handler, v2]
---

# MCP Gateway — Generic Handler (active) + V2 Dynamic Transport (demand-driven)

> **DRAFT — design under active discussion.** The **generic handler** is the active plan; the **V2 transport swap** is demand-driven/deferred. Not yet implemented.

**Status:** Generic handler = active plan. V2 transport swap = demand-driven / deferred. · **Owner:** Platform (MCP Platform)

Part of [MCP Development](../index.md). Related: [MCP Development Protocol](../steering/mcp-development-protocol.md), [MCP Platform Operations](../steering/mcp-platform-operations.md), [Design Alternatives & Incidents (INC-006)](./mcp-design-alternatives.md).

## 1. Why (two bugs, one root — and what's actually left to gain)

Two production incidents, same root cause: **the gateway kept its own copy of tool knowledge, separate from the MCP server, and it drifted.**

- **Stale static schema:** deploying a handler with new tools left the AgentCore target advertising the old set. Patched by `deploy-mcp.sh` Step 5b (schema sync).
- **Dispatch drift:** usain-bolt advertised 16 tools but the handler's hand-written chain routed only 7; the other 9 returned `Unknown tool`. Patched by a throwaway stopgap.

Both bugs are **already fixed in the current (V1) transport.** What remains worth doing:

**The generic handler (ACTIVE).** Replace every per-server hand-written dispatch chain with one generic handler that dispatches from the package's single-source `TOOLS` table (definition + `method`). This removes the dispatch-drift class at its root — permanently, not by discipline — and collapses N bespoke handlers into one. It runs on the CURRENT transport (AgentCore direct invoke), so the existing identity side-channel (`_user_email` injected into tool arguments by the proxy) keeps working unchanged.

### 1.1 V2 transport swap — DEMAND-DRIVEN, deferred

Switching the transport to **dynamic MCP-server targets over Lambda Function URLs (Streamable HTTP)** was the original "V2." Its remaining advantages once the generic handler ships are marginal or conditional:

- **Dynamic tool discovery** — only valuable when an MCP is deployed independently of the gateway. Today every tool change requires a GW action (build the Lambda layer), so the GW can refresh the schema then (Step 5b).
- **Protocol future-proofing** (prompts, resources, elicitation, streaming) — only valuable when our servers actually implement those capabilities. Today they are tools-only.
- **Clean user-identity propagation** — promoted from open question to a concrete V2 advantage (see §12 OQ3). V1 delivers no user identity to the handler, forcing the `_user_email`-in-arguments workaround; V2's Function-URL targets receive the real request incl. `Authorization`, letting the handler derive identity out-of-band.
- **Less AWS coupling / standards** — real but speculative.

**Triggers to revisit V2** (either): (1) **independent MCP deployment** — MCPs ship via their own pipelines without a GW action; (2) **a concrete need for a broader MCP feature** (prompts / resources / elicitation / streaming).

### 1.2 Can upgraded V1 achieve independent MCP deployment?

**No — not fully.** The coupling isn't in the dispatch code — it's in the **packaging topology**: an MCP's tool code is baked into a **GW-owned Lambda layer**. Options: **A (partial)** — the team's pipeline builds and publishes the layer version, handler references it (removes GW code changes, not the deploy topology); **B (true independence)** — the MCP runs as its own deployable unit and the generic handler *calls* it rather than embedding it, which **is exactly V2's shape**. So true independence ⇔ V2.

## 2. Principles (agreed)

1. **The MCP package stays pure.** A developer runs `server.py` locally as a real, unwrapped MCP server. The local server is the sacred artifact.
2. **All platform behavior lives in the GW repo, never in the package.**
3. **One tool truth, single-sourced.** The gateway holds no hand-maintained tool list; it dispatches from the package's `TOOLS` table.
4. `get-version` stays an ordinary package tool.
5. **All client methods are async.**
6. **Generic handler, per-MCP instances, uniform hooks.** Optional per-MCP hook modules for cross-cutting deviations. Hooks are uniform — never per-tool.

## 3. [DEMAND-DRIVEN / DEFERRED] Transport: Dynamic MCP Server Targets (the V2 mechanics)

> A retained blueprint, not the active plan. Implement only when a §1.1 trigger fires.

Replace static `toolSchema` AgentCore targets with **dynamic MCP Server targets** backed by **Lambda Function URLs**. Each deployed unit has a Lambda Function URL (stable, free, SigV4 auth), speaks MCP over Streamable HTTP (responds to `tools/list` and `tools/call`), and is registered as a DYNAMIC target — the gateway forwards `tools/list` per request, no cached static schema.

**What this retires:** static `toolSchema`; `deploy-mcp.sh` Step 5b schema sync; interceptor tool-definition enrichment/caching; per-server hand-written dispatch chains; and the `_user_email` arg-injection (identity moves to the forwarded request → `ctx`).

## 4. The generic handler (GW repo)

One generic implementation, instantiated per MCP (env var selects the package). Pure transport + discovery + dispatch — no tool-specific code beyond optional hooks. Serves `tools/list` from `TOOLS`, resolves the name (incl. compat routes), selects declared args, passes `ctx` **only** to `wants_context` tools, runs uniform hooks, calls the client method, returns the result.

## 5. The package contract

- **5.1** `TOOLS` — one list: definition + execution binding (`method`) + optional `wants_context`. Args derive from `inputSchema.properties`. `COMPAT_ROUTES` is package data.
- **5.2** `RequestContext` — package-owned shape (`identity`, extensible).
- **5.3** Client + factory — `create_client(*, token_provider=None)`.
- **5.4** `server.py` — real MCP server (local + deployed), generic registration loop, all async.

## 6. Context & extended args (the flexible contract)

Tool args come only from `inputSchema`. `RequestContext` is out-of-band, passed only to `wants_context` tools per-call (never shared mutable state → no async identity-bleed). Datadog-safe by construction: a tool without `wants_context` never receives `ctx`. Extensible: a new cross-cutting field is added to `RequestContext`, populated uniformly, zero impact on existing tools.

## 7. Hooks (GW repo, per-MCP, uniform)

Per-MCP hook modules in the GW repo; optional. Uniform, never per-tool (must not branch on tool name). Interface: `build_client()`, `build_context(request)`, `before(name, kwargs, request)`, `after(name, result)`. Identity is default behavior; the genuine per-MCP hooks today are env1's and chamberlain's `build_client` (DDB token provider remotely).

## 8. Per-server impact

- **usain-bolt (reference):** single-list `TOOLS`; remove `username` from `execute-run` schema, set `wants_context: True`; `execute_run` reads `ctx.identity`; no GW hook.
- **env1:** single-list bindings; no `wants_context` (needs a token-holding client); token provider via injectable — GW `build_client` hook supplies DDB provider remotely.
- **datadog, chamberlain:** single-list bindings, all async. datadog: no hooks. chamberlain: a `build_client` DDB-token hook like env1's.

## 9. Dev/Prod parity

A developer runs either the MCP server alone (real local dev, sacred) or the GW dispatcher in front of their local server (exercising the exact discovery + hook + dispatch path prod uses). Local == prod in code path; only the identity source differs.

## 10. Verification (definition of done per MCP)

- Every tool in `tools/list` is invoked (not just listed) and returns non-`Unknown tool`.
- Build-time contract test in the package makes "advertised but not executable" a build failure.
- Per-server auth/identity works; dev/prod parity holds for a sampled tool.

## 11. Migration Plan

1. usain-bolt = reference; verify all 16 invoked in PROD.
2. Ship the generic handler + hook framework in the GW repo.
3. Document the contract in the MCP Development Protocol.
4. env1 second — validates the `build_client` token-provider hook.
5. datadog, chamberlain — pure generic.
6. Retire V1 machinery (only if/when the V2 transport is adopted).

## 12. Open Questions — resolutions

- **OQ1 — Deployment unit: RESOLVED.** One generic-handler Lambda per MCP, own Function URL, own DYNAMIC target.
- **OQ2 — MCP-over-HTTP framing: RESOLVED.** DYNAMIC ListingMode; Lambda Function URLs are AWS-documented supported target hosts; Streamable HTTP.
- **OQ3 — Edge auth + user-identity propagation: RESOLVED (2026-09-01).** V1 (AgentCore direct-invoke) delivers no user identity to the handler — that is *why* the proxy injects `_user_email` into tool arguments (a V1 workaround, do not remove). V2's Function-URL target receives the real request incl. `Authorization`, so the handler derives identity out-of-band into `ctx.identity`. See [Design Alternatives → INC-006](./mcp-design-alternatives.md).
- **OQ4 — Hook loading: RESOLVED.** Convention (`gw_hooks/<package>.py`), default = none.
- **OQ5 — Local rehearsal: RESOLVED.** A `gw-dev` command running the dispatcher against a local server URL.
