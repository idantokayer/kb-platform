---
title: MCP Gateway — Headless / M2M Auth (Design)
description: Design for non-interactive (machine-to-machine) auth to the MCP gateway — Phase 1 specifies read-only M2M via gateway Cognito client-credentials; Phase 2 (writes via OBO) is an open spike.
tags: [plm, mcp, gateway, auth, m2m, headless, design]
---

# MCP Gateway — Headless / Machine-to-Machine Auth Design

**Status:** Draft — Phase 1 specified, Phase 2 is an open spike · **Owner:** Platform Tribe (MCP Platform)

Part of [MCP Development](../index.md). Related: [Remote MCP Gateway](./remote-mcp-gateway.md), [MCP Platform Operations](../steering/mcp-platform-operations.md), [MCP Development Protocol](../steering/mcp-development-protocol.md).

## 1. Context & Problem Statement

The gateway at `mcp.toluna.me` authenticates every caller through an **interactive OAuth authorization-code + PKCE** flow (browser → Cognito → Azure AD SSO → gateway JWT with the user's `email` claim). There is **no non-interactive path** — a CI runner, scheduled agent, or long-lived autonomous agent gets stuck at the interactive login.

### Driving requirement

Host a **full-featured KiroCrew agent on an EC2 instance**, fronting a **Microsoft Teams channel**, so R&D can interact with it in natural language and have it use our MCP tools. The EC2 agent is long-lived and headless (no browser), multi-user (many humans through one channel), and full-featured (eventually both read and write tools).

### The core tension: whose identity flows downstream?

Our platform carries **user identity as a first-class concept end-to-end** — the MCP is a thin client; the downstream service authorizes the user. A machine credential has **no user**. This forces the central split:

- **Reads** authorize on *presence of a valid gateway identity* → a machine (service-principal) identity is sufficient. **Buildable today.**
- **Writes** (env1 apply, chamberlain deploy) authorize on the *user's* role claim → require the **real user's** token to flow. A machine identity cannot satisfy this without breaking the "authorize the user, not the agent" invariant.

## 2. Auth Models Considered

| Model | Non-interactive? | Fits our email-claim model? | Per-user attribution? | Downstream authz | Verdict |
| --- | --- | --- | --- | --- | --- |
| Interactive OAuth (today) | No | Yes | Yes | User | Human-only; the gap |
| **A — M2M client-credentials** | Yes (native) | No user | No (agent principal) | Agent | **Phase 1 (reads)** |
| De-MFA'd human service user (`hiro@`) | Only via ROPC / bootstrapped refresh token | Yes | No (all = `hiro`) | User-shaped, blanket | **Rejected** |
| **B — OBO / token-exchange (RFC 8693)** | Yes (needs a user assertion) | Yes | Yes | **User** | **Phase 2 (writes) — spike** |

### Why the human service user was rejected

It does not remove the interactive-login blocker (only changes which non-interactive grant — ROPC, deprecated/blocked, or a fragile bootstrapped refresh token); requires disabling MFA / a Conditional-Access exception (a standing security hole); and gives the same lost-attribution as Model A anyway. An app identity (Model A) achieves the same "one stable agent identity" without a user, so MFA never applies.

## 3. Findings — What Is Verifiable About the Teams Runtime

- **KiroCrew's own headless auth is** `KIRO_API_KEY` — a single, process-wide static key carrying **no per-message user identity**. The runtime is Model A by nature.
- **The Teams integration is Bot Framework based.** The turn context exposes the sender's Azure AD object id / UPN — an identity *claim*, **not** an access token.
- **Obtaining an actual user AAD token** requires a separate user-SSO / OAuth consent flow; **there is no documentation that KiroCrew surfaces it** per turn.

**Consequence:** Model A is confirmed buildable; full OBO (Model B) is NOT confirmed buildable — it depends on an undocumented KiroCrew capability, which the Phase 2 spike must resolve.

## 4. Phase 1 — M2M Read-Only (Model A) — SPECIFIED

**Scope:** the EC2 agent authenticates as **itself** (a service principal) and may call **read-only, non-Pattern-A** profiles only: `datadog`, `usain-bolt` (reads), and the KB profiles. **No write tools. No Pattern-A (env1/chamberlain) profiles.**

- **4.1 Identity provider — gateway Cognito, nothing in Azure AD.** The M2M credential is issued by our own gateway Cognito pool (`investigation-agent-dev`, `us-east-1_aU7Vxiotq`, account 457416912276). Cognito mints client-credentials tokens itself; Azure AD federation plays no part. Nothing is created in Azure AD.
- **4.2 What gets created (all via Terraform):** a **resource server** (identifier `mcp-gateway`, scope `mcp-gateway/invoke`); an **M2M app client** with `AllowedOAuthFlows: ["client_credentials"]`, scoped narrowly, with a generated client secret; a **Secrets Manager** entry holding the `client_id`/`client_secret`. The agent obtains a token from the Cognito token endpoint (`POST /oauth2/token`, `grant_type=client_credentials`) and refreshes on expiry.
- **4.3 Gateway changes:** OAuth server accepts a Cognito-issued client-credentials token (validate via JWKS; distinguish from a user token); proxy maps it to a synthetic service-principal identity (`agent:kirocrew-teams@toluna.com`) and **restricts profiles** to a read-only / non-Pattern-A allow-list; interceptor / handlers unchanged.
- **4.4 Does NOT do:** no writes; no Pattern-A profiles; no per-user identity; no approval gate (an approval stamped with the agent's token is attribution, not delegation).
- **4.5 Verification (done = all green):** agent obtains a client-credentials token with no interactive step; a read tool call succeeds carrying the service-principal identity; a write or Pattern-A call on the M2M token is **rejected**; audit shows the synthetic actor string.

## 5. Phase 2 — OBO / Token-Exchange for Writes (Model B) — OPEN SPIKE

**Goal:** a write from Teams executes **as the real Teams user**, inheriting their role claims and permissions, audited as them — genuine **delegation** (RFC 8693), never impersonation.

**The spike question (blocking):** *Can the KiroCrew Teams runtime yield a verifiable user Azure AD token (or a user assertion exchangeable for one) per turn?* If yes → design real OBO (extend the Pattern-A `session:{email}:{profile}` model). If no → per-user permission inheritance is not achievable through this runtime; document the limitation.

**Explicitly not a Phase-2 fallback:** a half-delegation "approval + agent token" write path — it violates the authorization invariant.

## 6. Non-Gateway Systems (Bitbucket / Jira / Confluence)

The M2M Cognito credential authenticates the agent **to our gateway only**. Each external system needs its **own** scoped credential (Bitbucket access token; Jira/Confluence API token or OAuth app), belonging to the respective MCP (thin-client rule), stored in Secrets Manager. For writes, brokered per-user under the Phase-2 OBO conclusion.

## 7. Decisions & Open Questions

- **D1** — Identity provider for M2M is **gateway Cognito**, not Azure AD.
- **D2** — Human service user (`hiro@`) **rejected** in favor of an app identity.
- **D3** — Phase 1 is **read-only, non-Pattern-A**, single service-principal identity, **no approval gate**.
- **OQ1 (blocks Phase 2)** — Can the KiroCrew Teams runtime surface a per-user AAD token? Resolve by spike.
- **OQ2** — Synthetic actor string convention (`agent:<name>@toluna.com`) — confirm no collision with real audit consumers.
- **OQ3** — Does the gateway Cognito app client already permit a client-credentials grant + resource server, or must it be enabled?
