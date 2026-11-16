---
title: RTFM - Terraform Layers
description: How the RTFM stack maps onto the platform's shared, data and app Terraform layers — workspaces, resources per layer, cross-layer reads and the history of why resources moved.
tags: [plm, internal-tools, rtfm, terraform, layers]
---

# RTFM - Terraform Layers

Part of [RTFM](./index.md). General layer concepts are in [Terraform Infrastructure layers](../../terraform-infrastructure-layers.md).

All resources use the `rtfm-` prefix.

## Workspaces

| Layer | Workspaces | Runs | Accounts |
|---|---|---|---|
| shared | `shared-non-prod`, `shared-prod` | once per account | 525997793977 / 288513693345 |
| data | `qa-data`, `stg-data`, `dev-data`, `prod-data` | once per data workspace | same |
| app | `qa`, `stg`, `dev`, `idan`, `prod` … | once per env | same |

`module.config` (env1 config-manager) drives everything from env1 S3 JSON: `globals.environment` is `terraform.workspace`, `env_name` is `split("-", workspace)[0]`, `env_type` is `non-prod` or `prod`, and the AWS profile is `rtfm-<env_type>`.

## Shared layer — `terraform/shared/`

One set per account; long-lived.

- **Cognito** user pool, client and Hosted UI domain (one pool per account, see [Authentication](./authentication.md)).
- **SSM git credentials**: `/rtfm/spaces/ssh-key`, `/rtfm/spaces/github-pat`, `/rtfm/bb_at`.
- **microVM image build prerequisites**: S3 bucket `rtfm-<env_type>-microvm-artifacts` and IAM build role `rtfm-<env_type>-microvm-build`.
- Standard Monty skeleton: ECR, `source_pipeline_controller`, `test_framework`, `shared-security-group`, `switch-traffic` shared records, `datadog`, `finops`.

## Data layer — `terraform/data/`

One set per **data workspace**; must survive deploys. Several app envs can share one data workspace and are separated by the `env` dimension ([Spaces and Storage](./spaces-and-storage.md)).

- **DynamoDB** `rtfm-<data-workspace>-{state,routes,ws-connections}`. The hash key remains `space` / `connectionId` but the stored value is `<env>#<name>`.
- **EFS filesystem**, mount targets and security group, with VPC and subnets from `module.config.net`.
- **VPC egress network connector** so microVMs can reach EFS on NFS 2049.

## App layer — `terraform/app/`

Per env; safe to destroy and recreate.

- **PRIVATE REST API** front door (`apigw.tf`) with the greedy proxy route, reached through the shared execute-api VPC endpoint.
- **`switch-traffic` module** — private custom domain `<env_name>.<domain>`, private-zone Route53 record, RAM share and VPCE access association.
- **WebSocket API** with the `$connect` REQUEST authorizer (collab is disabled in practice).
- **Five control-plane Lambdas**: `rtfm-<env>-{proxy,lifecycle-manager,token-rotation,ws-collab,ws-authorizer}`.
- **EFS access points** rooted at `/spaces/<env>/<name>` and **seed items** keyed `<env>#<name>`.
- **EventBridge** reconcile and rotate schedules.
- Monty skeleton: `lambda_health`, REST API wiring and the CI/CD `pipeline` module.

## Cross-layer reads

- App reads **data** remote state (profile `rtfm-<env_type>`, workspace `<env>.data_workspace`) for table names/ARNs and EFS id/DNS.
- App computes **SSM parameter ARNs** from account and region (deterministic names) — no remote-state dependency on shared.
- App resolves **Cognito** by name through data sources.
- TokenRotation IAM (app) needs write access to the data-layer routes table.

## Not managed by Terraform

- **microVMs** are launched at runtime by LifecycleManager.
- **Per-space config** is data (env1 JSON and DynamoDB rows), not a deployed resource.
- **The `rtfm` microVM image** (`microvm-image:rtfm`) is published with `aws lambda-microvms create-microvm-image` / `update-microvm-image`, consuming the shared bucket and build role.

## Lambda build

Lambda zips are referenced at `terraform/app/.build/*.zip` (`source_code_hash` uses `try()` so validate and plan work before the zips exist). Build them from `service/control-plane`:

- Python (`proxy`, `lifecycle_manager`, `token_rotation`): `python build.py`. The proxy architecture is **x86_64** to match the lockfile's `awscrt` wheel.
- Node (`ws-authorizer`, `ws-collab`): `./build.sh`.

Wiring this build into the app pipeline buildspec is an open follow-up.

## Design history

- **Cognito:** per-env in app, then moved to shared as one per account.
- **DynamoDB:** per-env, then per-account, settled on per data workspace with an `env` column.
- **EFS:** filesystem per data workspace; env isolation through the access-point path.
- **Front door:** first a public HTTP API with public Route53/ACM, reworked to the platform-standard private REST API + VPC endpoint + `switch-traffic`.
- **Image name:** `rtfm-space` became `rtfm`.
