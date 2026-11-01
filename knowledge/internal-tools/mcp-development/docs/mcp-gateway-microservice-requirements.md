---
title: MCP Gateway — Microservice Promotion (Requirements, Draft)
description: Draft requirements to promote the hand-operated MCP gateway POC into a tribe-standard microservice with a CI/CD pipeline, moving the quality gate to PR review and deploys to merge-to-main.
tags: [plm, mcp, gateway, requirements, microservice, draft]
---

# MCP Gateway — Microservice Promotion Requirements

> **DRAFT / TBD — requirements under discussion, not yet scheduled.** A separate track from the Generic Handler / V2 work.

**Status:** Draft · **Owner:** Platform Tribe (MCP Platform)

Part of [MCP Development](../index.md). Related: [MCP Development Protocol](../steering/mcp-development-protocol.md), [MCP Platform Operations](../steering/mcp-platform-operations.md), [Monty template](../../../platform-microservice-templates/monty.md).

## 1. Context & Problem Statement

The remote MCP gateway at `mcp.toluna.me` is a mature, production POC serving 8 live profiles. It is **not** a Platform-tribe microservice — it runs on hand-rolled operations: deploys are executed **manually** via `deploy-mcp.sh` from a single operator's machine; only two people can deploy; the "quality gate" is a **human judgment call at deploy time**; infrastructure is applied by hand.

### The core tension

Two distinct things are bundled into one manual step: (1) **a quality gate** — human judgment on whether a change is sound (genuinely valuable); (2) **a deployment bottleneck** — a single operator is the only one who can ship (pure inefficiency). These are separable. The goal: **keep the decision human (at PR review) and make the mechanism self-serve (pipeline-driven)**.

### Why now

The Platform tribe already solves this exact problem with its microservice templates ([Monty](../../../platform-microservice-templates/monty.md)). The MCP gateway has been hand-simulating a pipeline; promoting it drops a bespoke POC onto rails that already exist.

## 2. Goals & Non-Goals

**Goals:** G1 remove the deployment bottleneck (deploys triggered by merge-to-main); G2 move the quality gate left (PR review + mechanical checks in the pipeline); G3 adopt tribe-standard operations (blue/green + rollback, Datadog, FinOps, multi-env); G4 base the service on the Monty template; G5 preserve all current capability; G6 decide the individual-MCP deployment-path question (§5).

**Non-Goals:** NG1 not rebuilding the MCP protocol layer (lift-and-shift); NG2 not changing the end-user experience; NG3 not migrating individual MCP package ownership; NG4 not solving the ENV1 v2 config gate.

## 3. Functional Requirements

- **FR1** — Pipeline-triggered deployment (merge to main → build → gates → deploy → traffic switch; no manual `terraform apply`/`deploy-mcp.sh`).
- **FR2** — Quality gates in the build (unit tests + SonarQube must pass; a red suite blocks the pipeline).
- **FR3** — Version drift guard as a pipeline stage.
- **FR4** — Preserve the gateway components (proxy, interceptor, OAuth server, handler Lambdas all deploy through the pipeline).
- **FR5** — Preserve AgentCore + KB infrastructure as Terraform in the app layer.
- **FR6** — Preserve auth (Azure AD SSO, per-profile Cognito delegation, OAuth tokens in DynamoDB, token refresh).
- **FR7** — Multi-environment support (dev/QA/staging/prod).
- **FR8** — MCP package consumption (gateway consumes MCP packages as pinned CodeArtifact dependencies; pipeline builds their Lambda layers).

## 4. Non-Functional Requirements

- **NFR1** — Reversibility (blue/green + automatic rollback on failed health checks).
- **NFR2** — Observability (Datadog APM on all gateway Lambdas).
- **NFR3** — Cost tracking (FinOps module).
- **NFR4** — Security continuity (secrets in Secrets Manager, IAM least-privilege, OAuth tokens in DynamoDB).
- **NFR5** — Zero-downtime migration (staged cutover).
- **NFR6** — Auditability (every deploy traceable to a merge/approval).

## 5. Key Decision: Individual-MCP Deployment Paths

Two layers can change independently: the gateway itself, and the individual MCP servers (each published to CodeArtifact and built into a Lambda layer).

### Option A — Single gateway pipeline (Phase 1 target)

The gateway is one microservice; MCP packages are its pinned dependencies. A new MCP version → bump the pinned version → gateway pipeline rebuilds that layer and deploys. Simplest to promote; quality gate at one place (the gateway PR); but MCP teams still depend on a gateway-repo change to ship.

### Option B — Per-MCP pipelines (Phase 2 vision)

Each MCP repo gets its own lightweight pipeline that publishes to CodeArtifact and builds its layer; the gateway consumes the new version automatically. Fully removes the central operator; but more pipelines, needs a safe pinning contract, and distributes the quality gate.

> **Cross-reference:** Option B — true independent MCP deployment — is architecturally the same decision as adopting the **V2 gateway transport** (see [Generic Handler & V2](./mcp-gateway-v2-design.md)). Phase 2 here and V2 there are one decision.

**Recommendation (to be confirmed):** Phase 1 = Option A; Phase 2 = Option B once the gateway service is stable.

## 6. Constraints & Assumptions

- **C1** — Base template: Monty (Python 3.12 Lambda), three-layer Terraform.
- **C2** — Gateway-as-one-service framing (Option A for Phase 1).
- **C3** — Existing repos stay; `remote-mcp-gateway` is the migration source.
- **C4** — Custom infrastructure gap: the gateway additionally owns AgentCore gateways, Bedrock KB connector targets, and OAuth/Cognito delegation — additive custom Terraform.
- **C5** — Multi-Lambda topology (proxy, interceptor, OAuth server, N handlers), unlike Monty's single Lambda.
- **C6** — AWS accounts: current infra in `datalab` (457416912276); decide whether to stay or adopt tribe-standard.

## 7. Migration Considerations

Multi-Lambda vs single-Lambda scaffold; AgentCore + KB Terraform authored into the app layer; cutover strategy (no downtime); per-MCP layer builds from pinned CodeArtifact versions.

## 8. Success Criteria

SC1 merge-to-main deploys to prod via pipeline; SC2 all profiles remain live and functionally identical; SC3 quality gates run automatically and can block a deploy; SC4 a failed deploy rolls back automatically; SC5 the human gate is at PR approval, not deploy; SC6 (meta) document how much of the promotion was achievable self-service.

## 9. Open Questions

OQ1 tribe ownership + app-role/app-type; OQ2 AWS account model; OQ3 Phase 2 pinning strategy; OQ4 does Monty's private API Gateway + IAM/SigV4 model fit the public SSO front door; OQ5 AgentCore/KB Terraform — fresh or existing tribe module; OQ6 formal new-service-creation vs existing-service promotion path.
