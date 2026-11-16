---
title: RTFM - Known Issues and Open Work
description: Outstanding RTFM work items, Terraform drift to reconcile, security follow-ups and the microVM concurrency limitation.
tags: [plm, internal-tools, rtfm, known-issues, backlog]
---

# RTFM - Known Issues and Open Work

Part of [RTFM](./index.md). Review and update this page whenever an item is closed.

## Prod rollout

- Run the full `shared-prod`, `prod-data` and `prod` Terraform: Cognito, SSM `/rtfm/bb_at`, tables, EFS, egress connector and the `prod#platform` seed item.
- Confirm the prod private zone is associated with the prod shared VPC (`platform-prod2`) before expecting DNS resolution.
- Launch the prod `platform` space.
- Seed the real `/rtfm/bb_at` token in prod.

## Terraform drift

- **Prod front door.** The PRIVATE custom domains, VPCE associations and RAM share were repaired manually. The live domains are not in Terraform state and stale ids may still be referenced. The next apply of the `app` / `prod` workspace can re-break the front door; run it deliberately and let `switch-traffic` recreate everything in one pass. See the troubleshooting section in [Deployment and Runbook](./deployment-and-runbook.md).
- The RAM share `rtfm_prod-custom_domain` still lists two dead domain ARNs (harmless; Terraform reconciliation clears them).
- `dev` has the same private-zone origin as qa. Apply the same private REST wiring and confirm its record points at the shared VPC endpoint.

## Configuration

- **env1 drift:** in `rtfm/app-env.json` the `idan` env's `platform` space still points at the old `platform.git` repo; qa, stg, dev and prod use `kb-platform.git`. Fix it if `idan` is used.

## Security follow-ups

- Rotate the `token` credential exposed in the OpenKnowledge origin repository's pushed history.
- Rotate the Bitbucket access token that was exposed in plaintext during development, then update SSM `/rtfm/bb_at` in **both** accounts.
- MCP endpoints are not behind Cognito (see [Architecture](./architecture.md)); consider an API-key gate at the proxy.
- Per-space `read_groups` authorization is not implemented.

## CI/CD

- Wire the Lambda zip build into the app pipeline buildspec; today it is a manual pre-apply step.
- Build and publish the `rtfm` microVM image in each target account through a repeatable process rather than by hand.

## microVM concurrency (429s)

A browser load fires about 40 concurrent requests (SPA chunks plus about 30 `/api/*` calls). The microVM platform has a low concurrent-request ceiling and sheds overflow with HTTP 429 (proven with API Gateway access logs: `integrationStatus=429`, `integrationLatency` about 190 ms).

Mitigations, in order of preference:

1. **Proxy-side asset cache** (implemented): immutable `/<space>/assets/*` served from the proxy Lambda without reaching the VM, plus 429 retry for idempotent requests.
2. **S3 asset origin** (cleaner long-term): publish build assets to S3 and add an API Gateway route `/{space}/assets/{proxy+}` straight to S3, so the microVM never sees asset traffic.

## Cosmetic

- `/api/sync/status` can report `"transport":"ssh"` for SSH-form remote URLs even though pulls run over HTTPS.
- Monty skeleton resources show non-destructive perpetual plan diffs.
