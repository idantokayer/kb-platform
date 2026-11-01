---
title: Module Dependency Graph
description: How Toluna's Terraform modules depend on each other — from configuration through to deployment — with per-module dependencies and an output-to-input mapping.
tags: [plm, terraform, modules, dependencies, architecture]
---

# Module Dependency Graph

How Toluna's Terraform modules depend on each other, from configuration through to deployment.

Part of [Terraform Modules](./index.md). See also [Terraform Infrastructure layers](../terraform-infrastructure-layers.md).

---

## Master Dependency Chain

```
                        ┌──────────────────────┐
                        │   config-manager      │
                        │   (every layer,       │
                        │    every service)      │
                        └──────────┬───────────┘
                                   │ config = module.config
                    ┌──────────────┼──────────────────────────┐
                    │              │                           │
                    ▼              ▼                           ▼
          ┌─────────────┐  ┌─────────────────┐  ┌──────────────────────┐
          │ SHARED LAYER│  │   APP LAYER     │  │    DATA LAYER        │
          └──────┬──────┘  └───────┬─────────┘  └──────────┬───────────┘
                 │                 │                        │
    ┌────────────┼────────┐       │              ┌─────────┼──────────┐
    │            │        │       │              │         │          │
    ▼            ▼        ▼       │              ▼         ▼          ▼
 pipeline-   switch-    ecr      │           dynamodb    sqs     s3 bucket
 controller  traffic             │
    │            │                │
    │            │     ┌──────────┼──────────────────┐
    │            │     │          │                   │
    │            │     ▼          ▼                   ▼
    │            │   lambda     ecs          frontend-pipeline
    │            │     │          │                   │
    │            │     ▼          ▼                   ▼
    │            │  lambda-    ecs-           S3 + CloudFront
    │            │  gitflow    gitflow        (deploy target)
    │            │     │          │
    │            │     ▼          ▼
    │            │  CodeDeploy  CodeDeploy
    │            │  (Lambda)   (ECS)
    │            │     │          │
    └────────────┴─────┴──────────┘
                       │
                       ▼
              merge buildspec calls
              switch-traffic to flip DNS
```

---

## Module-by-Module Dependencies

### 1. `config-manager` — The Root of Everything

**Consumed by:** Every other module in every layer. See [config-manager](./config-manager.md).

Every module receives `config = module.config` as its primary input. Config-manager reads S3 configuration buckets and Terraform remote state to produce a unified output object.

| Output Group | Consumers | What It Provides |
| --- | --- | --- |
| `globals` | All modules | `app_name`, `tribe`, `env_name`, `environment`, `aws_profile`, `env_vars` |
| `net` | `lambda`, `ecs`, `switch-traffic` | `vpc_id`, `private_subnets`, `public_subnets`, `vpce_alb`, `vpce_domain_name` |
| `app` | `lambda`, `ecs`, `*-gitflow`, `switch-traffic` | `pipeline_type`, `ecr_repo_url`, `image_uri`, `shared_security_group`, `lambda_create_common`, `ecs` settings |
| `app.lambda_pipeline` | `lambda-gitflow` | `from_env`, `pipeline_branch`, `run_integration_tests`, deploy hook config |
| `app.ecs_pipeline` | `ecs-gitflow` | Same as above, ECS-specific |
| `app.frontend_pipeline` | `frontend-pipeline` | SPA-specific pipeline config |
| `shared` | `switch-traffic`, `pipeline-controller` | `controller` config, DNS records, `bg_envs`, `lambdas_list`, `step_function_definition_list` |
| `data` | `dynamodb` module | `is_multitenant`, DynamoDB capacity, `toluna_envs` |
| `datadog` | `lambda`, `ecs` (via env vars) | `dd_api_key`, `dd_app_key`, `host_tags`, `dd_trace_sample_rate` |
| `ecr` | `lambda-gitflow`, `ecs-gitflow` | `repo_name`, `principal`, `replication_config` |

**Internal cross-layer reads (inside config-manager):**

- `modules/app` reads `terraform_remote_state.shared` → gets `ecr_repo_url`, `public_cert_arn`, `private_cert_arn`, `test_report_group`
- `modules/app` reads `terraform_remote_state.data` → gets `table_names`, `sqs_url_ssm_param_key`
- `modules/shared` reads app layer remote state → gets `lambdas_list`, `step_function_definition_list`, `app_event_source_mapping_list`

### 2. `pipeline-controller` (Shared Layer)

**Depends on:** [config-manager](./config-manager.md) → `config.shared.controller` for webhook config, environment lists.

**Consumed by:**

- [lambda-gitflow](./lambda-gitflow.md) / [ecs-gitflow](./ecs-gitflow.md) / [frontend-pipeline](./frontend-pipeline.md) (indirectly — the pipeline-controller's `pipeline_trigger` Lambda starts the CodePipeline created by these modules)
- [switch-traffic](./switch-traffic.md) (indirectly — the merge buildspec runs `terraform apply -target=module.switch-traffic`)

| Sub-module | Output | Consumer |
| --- | --- | --- |
| `pipeline_trigger` | Lambda ARN | S3 event notification triggers it when artifact uploaded |
| `merge_waiter` | Lambda ARN | CodeDeploy `BeforeAllowTraffic` hook invokes it |
| `bg_controller` | CodeBuild projects | Bitbucket webhooks trigger PR/merge builds |
| `notifier` | SNS topic ARN | CodePipeline state changes publish here |

### 3. `lambda` (App Layer)

**Depends on:** [config-manager](./config-manager.md) → `config.app.lambda_create_common`; existing API Gateway REST API; existing SQS queues (data layer, discovered via `data.aws_sqs_queue`).

**Consumed by:** [lambda-gitflow](./lambda-gitflow.md) (`function_list`), [switch-traffic](./switch-traffic.md), API Gateway deployment.

**Key output:** `aws_lambda_out` — a merged object containing the Lambda resource attributes plus `sns_integration_list` and `pipeline_type`.

### 4. `ecs` (App Layer)

**Depends on:** [config-manager](./config-manager.md) → `config.app.ecs`; ALB + target groups (inline `alb.tf`); ECR repository (via config-manager).

**Consumed by:** [ecs-gitflow](./ecs-gitflow.md) → `ecs_service_name`, `ecs_cluster_name`, `task_def_name`, `ecs_task_execution_role_arn`.

### 5. `lambda-gitflow` (App Layer)

**Depends on:** [config-manager](./config-manager.md) → `config.app.lambda_pipeline`; [lambda](./lambda.md) module outputs → `function_list`; S3 artifact bucket; ECR repository; `merge_waiter` or `test-framework-manager` Lambda.

**Consumed by:** [pipeline-controller](./pipeline-controller.md)'s `pipeline_trigger` Lambda; merge buildspec calls [switch-traffic](./switch-traffic.md).

### 6. `ecs-gitflow` (App Layer)

**Depends on:** [config-manager](./config-manager.md) → `config.app.ecs_pipeline`; [ecs](./ecs.md) module outputs; ALB listener ARNs (443 prod, 4443 test); ALB target group names (blue/green); S3 artifact bucket, ECR repository, merge_waiter/test-framework Lambda.

### 7. `switch-traffic` (Shared + App Layers)

**Depends on:** [config-manager](./config-manager.md) → `config.shared.*` and `config.net.*`; NET account AWS provider (RAM sharing / VPC endpoint associations); existing Route53 zones, ACM certs, API Gateways, ALBs, SQS queues, SNS topics.

**Consumed by:** [pipeline-controller](./pipeline-controller.md)'s merge buildspec; all services (DNS routing).

### 8. `ecr` (Shared Layer)

**Depends on:** [config-manager](./config-manager.md) → `config.ecr`.

**Consumed by:** [config-manager](./config-manager.md) (app sub-module) reads shared remote state for `ecr_repo_url`; [lambda-gitflow](./lambda-gitflow.md) / [ecs-gitflow](./ecs-gitflow.md) push/pull images.

---

## Output → Input Mapping Table

| Producing Module | Output | Consuming Module | Input |
| --- | --- | --- | --- |
| `config-manager` | `config` (full object) | Every module | `config` or `var.config` |
| `config-manager.app` | `ecr_repo_url` | `lambda`, `ecs`, `*-gitflow` | Image URI base |
| `config-manager.app` | `lambda_create_common` | `lambda` | Default Lambda settings |
| `config-manager.app` | `shared_security_group` | `lambda`, `ecs` | VPC security group |
| `config-manager.app` | `pipeline_type` | `*-gitflow`, `switch-traffic` | CI/CD/dev behavior |
| `config-manager.shared` | `lambdas_list` | `switch-traffic` (shared) | Lambda event source mappings, SNS subs |
| `config-manager.shared` | `step_function_definition_list` | `switch-traffic` (shared) | Step Function creation |
| `config-manager.shared` | `controller` | `pipeline-controller` | Webhook + pipeline config |
| `config-manager.shared` | `app_records`, `ecs_records`, `spa_records` | `switch-traffic` (shared) | DNS record creation |
| `config-manager.data` | `is_multitenant`, `toluna_envs` | `dynamodb` | Per-tenant table creation |
| `lambda` | `aws_lambda_out` | `lambda-gitflow` | `function_list` for CodeDeploy |
| `lambda` | API GW integration output | `aws_api_gateway_deployment` | Deployment trigger |
| `ecs` | `ecs_service_name`, `ecs_cluster_name` | `ecs-gitflow` | CodeDeploy target |
| `ecs` | `ecs_task_execution_role_arn` | `ecs-gitflow` | IAM for CodeDeploy |
| `ecr` | `ecr_repo_url` (via remote state) | `config-manager.app` | Image URI computation |
| `pipeline-controller.pipeline_trigger` | Lambda ARN | S3 event notification | Pipeline start |
| `pipeline-controller.merge_waiter` | Lambda ARN | CodeDeploy hooks | Deploy gate |
| `pipeline-controller.bg_controller` | CodeBuild projects | Bitbucket webhooks | PR/merge builds |
| `switch-traffic` | DNS records | Route53 | Traffic routing |
| `switch-traffic` | `live` alias mappings | Lambda/SFN consumers | Blue/green target |
| `test-framework` | Report group ARNs (via remote state) | `*-gitflow` CodeBuild projects | Test reporting |
| `shared-security-group` | SG ID (via remote state) | `config-manager.app` | `shared_security_group` |

---

## Dependency Order for Initial Setup

When setting up a new service from scratch, layers and modules must be applied in this order:

```
1. DATA LAYER
   └── terraform workspace select {env}-data
   └── terraform apply
   └── Creates: DynamoDB tables, SQS queues, S3 buckets, SSM params

2. SHARED LAYER (shared-non-prod first, then shared-prod)
   └── terraform workspace select shared-non-prod
   └── terraform apply
   └── Creates: ECR repo, pipeline controller, test framework,
                shared security group, DNS records, FinOps

3. APP LAYER (per environment)
   └── terraform workspace select {env}  (e.g., qa, stg, prod)
   └── terraform apply
   └── Creates: Lambda/ECS functions, API Gateway, ALB,
                CodePipeline, CodeDeploy, switch-traffic (dev only)
```

The shared layer must exist before the app layer because:

- App layer reads shared remote state for ECR URL, certificates, test report groups
- Pipeline controller (shared) must exist to trigger pipelines (app)
- Shared security group must exist for Lambda/ECS VPC config

The data layer must exist before the app layer because:

- App layer reads data remote state for DynamoDB table names, SQS URLs
- Lambda env vars reference data layer resource names
