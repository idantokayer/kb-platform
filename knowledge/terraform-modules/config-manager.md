---
title: config-manager
description: The central configuration hub for all Toluna Terraform modules — the root of the module dependency tree that resolves platform context from the workspace name.
tags: [plm, terraform, module, config-manager, shared, app, data]
---

# config-manager

The central configuration hub for all Toluna Terraform modules. Every module in every layer receives `config = module.config` as its primary input.

Part of [Terraform Modules](./index.md). See the [Module Dependency Graph](./module-dependency-graph.md) and [Terraform Infrastructure layers](../terraform-infrastructure-layers.md).

---

## Quick Reference

| Property | Value |
| --- | --- |
| **Registry** | `env1.toluna.me/toluna/terraform-toluna-config-manager/aws` |
| **Current Version** | 3.1.5 |
| **Repository** | [tolunaengineering/terraform-toluna-config-manager](https://bitbucket.org/tolunaengineering/terraform-toluna-config-manager) |
| **Present In** | Shared, App, Data (all layers) |

---

## Purpose

Config-manager is the **root of the module dependency tree**. It reads workspace name, S3 configuration buckets, and Terraform remote state to produce a unified output object that every other module consumes. It eliminates manual variable passing — all platform context (environment, VPC, profiles, certificates, ECR URLs) is resolved automatically from the workspace name.

It creates **no AWS resources** — it's pure computation and data lookups.

---

## Runtime Behavior

Config-manager runs only at `terraform plan/apply` time. It:

1. Detects the current layer from workspace name (`shared-*` → shared, `*-data` → data, everything else → app)
2. Reads S3 config bucket (`toluna-{tribe}-tf-config-data/{app_name}/`) to load `app-env.json`, `shared-env.json`, `data-env.json`
3. Strips blue/green color suffixes to compute `data_env_name` (so `qa-blue` resolves to `qa` for data layer lookups)
4. Reads remote state from other layers (app reads shared+data, shared reads app) to resolve cross-layer outputs like ECR URL, Lambda lists, certificates
5. Resolves networking (VPC ID, subnets, VPC endpoint) from platform state bucket
6. Exposes everything as a single `config` output object consumed by all other modules

---

## Module Interactions

### Outputs Consumed By (everything)

| Consumer | What It Uses |
| --- | --- |
| [lambda](./lambda.md) | `config.app.lambda_create_common`, `config.net.vpc_id`, `config.net.private_subnets`, `config.app.shared_security_group` |
| [lambda-gitflow](./lambda-gitflow.md) | `config.app.pipeline_type`, `config.app.pipeline_branch`, `config.app.ecr_repo_url` |
| [pipeline-controller](./pipeline-controller.md) | `config.shared.controller`, `config.globals.app_envs` |
| [switch-traffic](./switch-traffic.md) | `config.shared.lambdas_list`, `config.net.domain_association_role_arn`, `config.shared.app_records` |
| [ecr](./ecr.md) | `config.ecr.repo_name`, `config.ecr.principal` |
| [dynamodb](./dynamodb.md) | `config.data.is_multitenant`, `config.globals.env_vars.toluna_envs` |
| [test-framework](./test-framework.md) | `config.globals`, `config.net` (VPC for test runner) |
| [finops](./finops.md) | `config.globals` (app_name, tribe for tagging) |

### Cross-Layer Remote State Reads (internal)

| Sub-module | Reads From | Data Retrieved |
| --- | --- | --- |
| `modules/app` | Shared remote state | `ecr_repo_url`, `public_cert_arn`, `private_cert_arn`, `test_report_group`, `shared_security_group` |
| `modules/app` | Data remote state | `table_names`, `sqs_url_ssm_param_key` |
| `modules/shared` | App remote state | `lambdas_list`, `step_function_definition_list`, `app_event_source_mapping_list` |

---

## Inputs

| Variable | Type | Required | Description |
| --- | --- | --- | --- |
| `app_name` | string | Yes | Application name (e.g., `candynet-release`) |
| `app_type` | string | Yes | Service type: `sam` (Lambda), `ecs`, `spa`, `glue` |
| `tribe` | string | Yes | Tribe name (e.g., `buffet`) |
| `app_role` | string | Yes | Role for RBAC (e.g., `EXP`) |
| `enable_rbac` | bool | No | Enable role-based access (default: `false`) |
| `dd_trace_sample_rate_prod` | string | No | Datadog trace sample rate for prod (default: `0.01`) |
| `dd_trace_sample_rate_non_prod` | string | No | Datadog trace sample rate for non-prod (default: `1`) |

---

## Key Outputs

| Output Group | Key Fields | Description |
| --- | --- | --- |
| `globals` | `app_name`, `environment`, `env_name`, `env_type`, `aws_profile`, `tribe`, `app_envs`, `env_vars` | Core identity and environment context |
| `net` | `vpc_id`, `private_subnets`, `public_subnets`, `vpce_alb`, `domain_association_role_arn` | Networking |
| `app` | `pipeline_type`, `ecr_repo_url`, `shared_security_group`, `lambda_create_common`, `private_cert_arn` | App-layer specifics (only populated in app layer) |
| `shared` | `controller`, `lambdas_list`, `step_function_definition_list`, `bg_envs` | Shared-layer specifics |
| `data` | `is_multitenant`, `toluna_envs` | Data-layer specifics |
| `datadog` | `dd_api_key`, `dd_app_key`, `dd_trace_sample_rate`, `host_tags` | Observability config |
| `ecr` | `repo_name`, `principal`, `replication_config` | ECR settings |

---

## Sub-modules

| Sub-module | Purpose |
| --- | --- |
| `modules/app` | App-layer outputs: reads shared+data remote state, resolves pipeline config, ECR URL, certificates |
| `modules/shared` | Shared-layer outputs: reads app remote state, resolves Lambda lists, controller config |
| `modules/data` | Data-layer outputs: multi-tenant config, DynamoDB settings |
| `role_base_access` | External module (`terraform-aws-role-based-access`) — API key management for RBAC |

---

## Usage Example

```hcl
module "config" {
  source  = "env1.toluna.me/toluna/terraform-toluna-config-manager/aws"
  version = "3.1.5"

  providers = {
    aws            = aws
    aws.s3_backend = aws.s3_backend
  }

  app_name    = "my-service-release"
  tribe       = "buffet"
  app_role    = "EXP"
  app_type    = "sam"
  enable_rbac = false
}
```

The `aws.s3_backend` provider is always the non-prod profile (used for reading S3 config bucket regardless of which environment you're applying to).
