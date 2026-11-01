---
title: Terraform Infrastructure layers
description: How Toluna's Terraform architecture splits every microservice into shared, app, and data layers — each with its own state, workspace, and apply lifecycle.
tags: [plm, terraform, layers, architecture, state, blue-green]
---

# Infrastructure Layers

Toluna's Terraform architecture splits every microservice into three independent layers: shared, app, and data. Each layer has its own state file, workspace, and apply lifecycle. **Each workspace in each layer represents a dedicated, isolated Terraform state** — ensuring complete separation between environments and deployment slots.

Part of [VII - Platform Infrastructure](./platform-infrastructure.md). See also the [Module Dependency Graph](./terraform-modules/module-dependency-graph.md) and [config-manager](./terraform-modules/config-manager.md), which resolves the layer from the workspace name.

---

## Why Three Layers?

The separation solves three problems:

1. **Blast radius isolation** — Destroying an app layer (during blue/green cleanup) never touches persistent data (DynamoDB, SQS) or shared infrastructure (ECR, DNS, pipelines).
2. **Lifecycle independence** — Data resources (tables, queues) are created once and rarely change. App resources (Lambda functions, ECS services) change on every deployment. Shared resources (ECR, pipeline controller) change only when the platform evolves.
3. **Cross-environment coordination** — The shared layer manages resources that span all environments (DNS records, pipeline webhooks, ECR repos), while app and data layers are per-environment.

---

## Layer Classification

| Layer | Mutability | Scope | Destroyed During Deployment? |
| --- | --- | --- | --- |
| **App** | Mutable | Per environment | **Yes** — blue/green workspaces are created and destroyed on every deployment |
| **Data** | Immutable/Persistent | Per environment | **No** — persists across all deployments. One workspace per environment (e.g., `qa-data`, `stg-data`, `prod-data`, `{dev}-data`) |
| **Shared** | Immutable | Per AWS account | **No** — account-wide infrastructure that never gets destroyed during normal operations |

### App Layer — Mutable (destroyed each deployment)

Contains resources that are **created and destroyed** as part of every blue/green infrastructure deployment cycle. When a PR triggers a pipeline, a new app workspace (e.g., `qa-blue` or `qa-green`) is created with fresh infrastructure. After merge and traffic switch, the old workspace is destroyed.

This is intentional — app resources are stateless compute that can be recreated at any time without data loss.

### Data Layer — Persistent/Immutable per environment

Contains **stateful resources that must survive deployments**: databases, queues, buckets. Each environment has its own dedicated workspace and state (e.g., `qa-data`, `stg-data`, `prod-data`, `waseem-data`). These workspaces are created once and rarely modified.

The blue/green mechanism never touches the data layer — both `qa-blue` and `qa-green` app workspaces share the same `qa-data` tables and queues ([config-manager](./terraform-modules/config-manager.md) strips the color suffix to resolve the correct data workspace).

### Shared Layer — Immutable, account-wide

Contains **account-scoped infrastructure** that serves all environments and is never destroyed during normal operations. Only two workspaces exist: `shared-non-prod` and `shared-prod`.

Shared layer modules include:

- [terraform-aws-ecr](./terraform-modules/ecr.md) — Docker image registry, one per service across all environments
- [terraform-aws-pipeline-controller](./terraform-modules/pipeline-controller.md) — Bitbucket webhooks, merge waiter, rollback controller, pipeline trigger
- [terraform-toluna-switch-traffic](./terraform-modules/switch-traffic.md) — DNS records, event source mappings, EventBridge schedules, SNS subscriptions
- [terraform-aws-codepipline-test-framework](./terraform-modules/test-framework.md) — Integration test runner and report groups
- [terraform-aws-shared-security-group](./terraform-modules/shared-security-group.md) — Common VPC security group (kept here to avoid slow AWS networking API calls during app layer create/destroy)
- [terraform-aws-finops](./terraform-modules/finops.md) — Budget alerts and cost anomaly detection
- [terraform-aws-datadog-integration](./terraform-modules/datadog-integration.md) — AWS↔Datadog integration (one per account)
- AppConfig — Feature flag application and environments (inline resources)
- [terraform-aws-event-driven-communication](./terraform-modules/event-driven-communication.md) — Cross-tribe EventBridge bus and SNS topic

---

## Layer Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    SHARED LAYER (Immutable)                      │
│  Workspace: shared-non-prod, shared-prod                        │
│  Scope: One per AWS account                                     │
│  State: s3://{tribe}-tf-state-data/terraform/{app}/shared       │
│                                                                 │
│  Contains: ecr, pipeline-controller, switch-traffic (DNS),      │
│            test-framework, shared-security-group, finops,        │
│            AppConfig, datadog-integration, event-driven-comm     │
└─────────────────────────────────────────────────────────────────┘
                              │
                    config-manager reads
                    remote state bidirectionally
                              │
┌─────────────────────────────┴───────────────────────────────────┐
│                    APP LAYER (Mutable)                           │
│  Workspace: qa, qa-blue, qa-green, stg, prod, prod-blue, ...   │
│  Scope: One per environment per deployment slot                 │
│  State: s3://{tribe}-tf-state-data/terraform/{app}/app          │
│                                                                 │
│  Contains: lambda, ecs, CloudFront, API Gateway, ALB,           │
│            lambda-gitflow / ecs-gitflow, switch-traffic (dev)    │
│  Destroyed and recreated on every blue/green deployment         │
└─────────────────────────────────────────────────────────────────┘
                              │
                    config-manager reads
                    remote state
                              │
┌─────────────────────────────┴───────────────────────────────────┐
│                    DATA LAYER (Persistent)                       │
│  Workspace: qa-data, stg-data, prod-data, {dev}-data            │
│  Scope: One per environment                                     │
│  State: s3://{tribe}-tf-state-data/terraform/{app}/data         │
│                                                                 │
│  Contains: dynamodb, SQS queues, S3 buckets, SSM parameters     │
│  Never destroyed during deployments                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Workspace Naming Convention

Config-manager auto-detects the layer from the Terraform workspace name:

```hcl
local.layer = startswith(terraform.workspace, "shared-") ? "shared"
            : endswith(terraform.workspace, "-data")     ? "data"
            : "app"
```

| Workspace Pattern | Layer | Example Workspaces |
| --- | --- | --- |
| `shared-*` | shared | `shared-non-prod`, `shared-prod` |
| `*-data` | data | `qa-data`, `stg-data`, `prod-data`, `waseem-data` |
| Everything else | app | `qa`, `qa-blue`, `qa-green`, `stg`, `prod`, `prod-blue`, `waseem` |

**Each workspace = a dedicated Terraform state file.** This means `qa-blue` and `qa-green` have completely separate state, and destroying one has zero impact on the other or on any data/shared resources.

### Blue/Green Workspace Suffixes

App layer workspaces use color suffixes for blue/green deployments:

| Workspace | Purpose |
| --- | --- |
| `qa` | Base QA environment (used for config resolution) |
| `qa-blue` | Blue deployment slot |
| `qa-green` | Green deployment slot |
| `prod` | Base prod environment |
| `prod-blue` | Blue deployment slot |
| `prod-green` | Green deployment slot |
| `waseem` | Developer overlay (no blue/green) |

Config-manager strips the color suffix to compute `data_env_name` — so `qa-blue` and `qa-green` both resolve to `qa` for data layer lookups. This ensures both colors share the same DynamoDB tables and SQS queues.

---

## State Isolation

Each workspace in each layer has its own **completely independent Terraform state file** stored in S3:

```
s3://toluna-{tribe}-tf-state-data/
  └── terraform/
      └── {app_name}/
          ├── shared    ← shared layer state (workspace-prefixed)
          │   ├── shared-non-prod.tfstate
          │   └── shared-prod.tfstate
          ├── app       ← app layer state (workspace-prefixed)
          │   ├── qa.tfstate
          │   ├── qa-blue.tfstate
          │   ├── qa-green.tfstate
          │   ├── prod-blue.tfstate
          │   └── waseem.tfstate
          └── data      ← data layer state (workspace-prefixed)
              ├── qa-data.tfstate
              ├── stg-data.tfstate
              └── prod-data.tfstate
```

This means:

- Destroying `qa-blue` app workspace only affects resources in `qa-blue.tfstate` — nothing else
- The `qa-data.tfstate` is shared by both `qa-blue` and `qa-green` (they read its outputs via remote state)
- The `shared-non-prod.tfstate` is referenced by ALL app workspaces in the non-prod account

---

## Cross-Layer Data Flow

### How Layers Discover Each Other

Layers communicate through Terraform remote state stored in S3:

```
State bucket: s3://toluna-{tribe}-tf-state-data/terraform/{app_name}/{layer}

Shared state: s3://toluna-{tribe}-tf-state-data/terraform/{app_name}/shared
App state:    s3://toluna-{tribe}-tf-state-data/terraform/{app_name}/app
Data state:   s3://toluna-{tribe}-tf-state-data/terraform/{app_name}/data
```

Config-manager's sub-modules read remote state from other layers:

| Reader | Reads From | Data Retrieved |
| --- | --- | --- |
| `config.app` (app sub-module) | Shared remote state | `ecr_repo_url`, `public_cert_arn`, `private_cert_arn`, `test_report_group`, `coverage_report_group`, `dev_keys_to_add_to_plan` |
| `config.app` (app sub-module) | Data remote state | `table_names`, `sqs_url_ssm_param_key` |
| `config.shared` (shared sub-module) | App remote state | `lambdas_list`, `step_function_definition_list`, `app_event_source_mapping_list` |

### Flow During Blue/Green Deployment

```
1. DATA LAYER (pre-existing, never destroyed)
   │  DynamoDB tables, SQS queues exist in qa-data workspace
   │  Outputs: table_names, sqs_url
   │
   ▼
2. APP LAYER (PR build creates new blue/green workspace)
   │  Creates Lambda functions / ECS service in qa-blue workspace
   │  Reads data layer outputs for env vars
   │  Reads shared layer outputs for ECR URL, certs
   │  Outputs: lambdas_list, function ARNs
   │
   ▼
3. SHARED LAYER (merge build runs switch-traffic)
   │  Reads app layer outputs to discover new functions
   │  Creates/updates DNS records pointing to new functions
   │  Creates/updates event source mappings for new `live` aliases
   │  Creates/updates EventBridge schedules, SNS subscriptions
   │
   ▼
4. APP LAYER (merge build destroys old workspace)
   │  Old blue/green workspace state destroyed (e.g., qa-green.tfstate)
   │  Old Lambda versions / ECS task sets removed
   │  Data layer and shared layer completely untouched
```

---

## Configuration Sources (S3)

Each layer reads its configuration from S3 JSON files managed via env1:

```
s3://toluna-{tribe}-tf-config-data/
  └── {app_name}/
      ├── app-env.json      ← App layer config
      ├── shared-env.json   ← Shared layer config
      └── data-env.json     ← Data layer config
```

### app-env.json Structure

Contains per-environment settings keyed by workspace name:

```json
{
  "prod": {
    "tribe": "buffet",
    "shared_vpc_env": "prod2",
    "env_type": "prod",
    "ecs_service_desired_count": "3",
    "pipeline_branch": "master",
    "pipeline_type": "cd",
    "from_env": "qa",
    "is_managed_env": true,
    "run_integration_tests": true,
    "data_workspace": "prod-data"
  },
  "stg": {
    "tribe": "buffet",
    "shared_vpc_env": "stg2",
    "env_type": "non-prod",
    "ecs_service_desired_count": "3",
    "pipeline_branch": "staging",
    "pipeline_type": "cd",
    "from_env": "prod",
    "is_managed_env": true,
    "run_integration_tests": true,
    "data_workspace": "stg-data"
  },
  "qa": {
    "tribe": "buffet",
    "shared_vpc_env": "qa2",
    "env_type": "non-prod",
    "ecs_service_desired_count": "3",
    "pipeline_branch": "develop-v2",
    "pipeline_type": "ci",
    "from_env": "qa",
    "is_managed_env": true,
    "run_integration_tests": true,
    "run_stress_tests": false,
    "data_workspace": "qa-data"
  },
  "dev": {
    "tribe": "buffet",
    "shared_vpc_env": "qa2",
    "env_type": "non-prod",
    "ecs_service_desired_count": "2",
    "pipeline_branch": "devops",
    "pipeline_type": "dev",
    "from_env": "devops",
    "is_managed_env": false,
    "run_integration_tests": false,
    "data_workspace": "dev-data"
  }
}
```

### shared-env.json Structure

Contains account-level settings (domain, AWS account IDs, budgets, cost anomaly thresholds, Datadog config, app type). Keyed by `shared-prod` and `shared-non-prod`.

### data-env.json Structure

Contains data layer settings (tribe, VPC env, AWS profile, allowed environments, DynamoDB capacity/autoscaling, datalake export schedule). Keyed by `{env}-data`.

---

## Layer Interaction During Common Operations

### Adding a New Environment

1. Add environment to `app-env.json` in S3 (via env1)
2. Add environment to `shared-env.json` `app_envs` list
3. Add `{env}-data` entry to `data-env.json`
4. Apply data layer: `terraform workspace new {env}-data && terraform apply`
5. Apply shared layer: `terraform apply` (picks up new env from config)
6. Apply app layer: `terraform workspace new {env} && terraform apply`

### Destroying an Environment

1. Destroy app layer: `terraform workspace select {env} && terraform destroy`
2. Destroy data layer: `terraform workspace select {env}-data && terraform destroy`
3. Remove from S3 config files
4. Apply shared layer to clean up DNS records

### Adding a New DynamoDB Table

1. Add table definition in `terraform/data/dynamodb.tf`
2. Apply data layer: `terraform apply`
3. Update Lambda env vars in `terraform/app/` to reference new table name
4. Apply app layer: `terraform apply`

No shared layer changes needed — data and app are independent.
