---
title: RTFM - Read-only Knowledge-Base Platform
description: Git-backed Markdown knowledge bases ("spaces") published read-only to browsers and AI agents over MCP, served by on-demand per-space microVMs behind API Gateway + Cognito.
tags: [plm, internal-tools, rtfm, knowledge-base, mcp, index]
---

# RTFM - Read-only Knowledge-Base Platform

RTFM publishes git-backed Markdown knowledge bases ("spaces") **read-only** to the browser and to AI agents over **MCP**. Each space is a git repository rendered as a navigable wiki. Compute is on-demand: a space's Firecracker microVM boots on first request and suspends when idle.

RTFM is built on OpenKnowledge and is the platform behind Toluna's knowledge bases. Git is the source of truth; the browser is a read-only view.

Part of [Internal Tools](../index.md).

## At a glance

| Field | Value |
|---|---|
| Service name | `rtfm` |
| Jira ticket | [PLM-4101](https://toluna.atlassian.net/browse/PLM-4101) |
| Tribe | Platform |
| Compute | Lambda (Python / Node) + API Gateway + Lambda MicroVMs |
| Owner | idan.tokayer |
| Source repo | `tolunaengineering/rtfm` (Bitbucket) |
| Design doc | [RTfM - Design](https://toluna.atlassian.net/wiki/spaces/PLM/pages/3055550488/RTfM+-+Design) |

## AWS accounts and domains

| Environment | Account | ID | Domain | AWS SSO profile |
|---|---|---|---|---|
| Non-prod (dev / qa / stg) | Rtfm-Non-Prod | 525997793977 | `<env>.rtfm-non-prod.toluna.me` | `rtfm-non-prod` |
| Prod | Rtfm-Prod | 288513693345 | `<env>.rtfm.toluna.me` | `rtfm-prod` |

Region: `us-east-1`.

## What each child page covers

- [Architecture](./architecture.md) — components, trust boundaries, design decisions and caveats.
- [Request Routing](./request-routing.md) — front door, proxy Lambda, injected headers, status-code reading guide.
- [Control Plane](./control-plane.md) — LifecycleManager (launch and reconcile), scheduling, token rotation.
- [Authentication](./authentication.md) — Cognito login, internal proxy tokens, per-space git credentials.
- [Spaces and Storage](./spaces-and-storage.md) — how spaces are configured in env1, the `env` dimension, EFS persistence.
- [Terraform Layers](./terraform-layers.md) — what lives in `shared`, `data` and `app`, and why.
- [Deployment and Runbook](./deployment-and-runbook.md) — deploy order, DNS, current deployment state, operational procedures.
- [Cost Estimate](./cost-estimate.md) — monthly cost model per space.
- [Release Notes v1.0](./release-notes-v1.0.md) — what shipped in v1.0 and known limitations.
- [Known Issues and Open Work](./known-issues-and-open-work.md) — outstanding items and drift to reconcile.

## How to use

- **Read a space:** open `https://<env>.<domain>/<space>/` (e.g. `https://qa.rtfm-non-prod.toluna.me/platform/`) and sign in with Cognito.
- **Agents (MCP):** `POST /<space>/mcp` (StreamableHTTP) on the same host.
- **Add or change a space:** edit the env1 `rtfm/app-env.json` entry — see [Spaces and Storage](./spaces-and-storage.md).
- **Troubleshoot a failing request:** start with the status-code table in [Request Routing](./request-routing.md).
