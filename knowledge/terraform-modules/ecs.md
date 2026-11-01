---
title: ecs
description: Creates an ECS Fargate service with a Datadog sidecar, IAM roles, CloudWatch logs, and optional autoscaling, supporting CodeDeploy blue/green via ALB target group switching.
tags: [plm, terraform, module, ecs, fargate, app-layer]
---

# ecs

Creates an ECS Fargate service with a Datadog sidecar container, IAM roles, CloudWatch log groups, and optional autoscaling (CPU-based or ALB request-based). Supports CodeDeploy blue/green deployment via ALB target group switching.

Part of [Terraform Modules](./index.md). Deployed by [ecs-gitflow](./ecs-gitflow.md).

---

## Quick Reference

| Property | Value |
| --- | --- |
| **Registry** | `env1.toluna.me/toluna/terraform-aws-ecs/aws` |
| **Current Version** | 3.0.2 |
| **Repository** | [tolunaengineering/terraform-aws-ecs](https://bitbucket.org/tolunaengineering/terraform-aws-ecs) |
| **Present In** | App layer |

---

## Purpose

The standard way to create ECS Fargate services on the platform. Handles cluster creation, task definition (app container + Datadog sidecar), Fargate service, IAM roles, and optional autoscaling. Works with an external ALB (defined in the service's `alb.tf`) and CodeDeploy (managed by `ecs-gitflow`).

---

## Runtime Behavior

### At terraform apply time

1. **Resolves current image** — runs a shell script (`get_container_image.sh`) that looks up the correct Docker image tag in ECR. Falls back to `latest` → `soa-base` if no image exists yet (first deployment)
2. **Renders container definitions** — generates JSON with app container + Datadog sidecar, automatically calculating memory/CPU allocation between them
3. **Creates ECS cluster** — one cluster per service: `{app_name}-{environment}`
4. **Creates task definition** — Fargate-compatible, with both containers, environment variables, secrets, and log configuration
5. **Creates ECS service** — with `CODE_DEPLOY` deployment controller (when ALB is provided), referencing the blue target group
6. **Creates IAM role** — task execution role with ECS, SSM, CloudWatch, and optional Datadog permissions
7. **Optionally creates autoscaling** — based on CPU utilization or ALB request count

### Important lifecycle behavior

The service has `lifecycle { ignore_changes = [task_definition, load_balancer] }`. This means after initial creation, **CodeDeploy manages the task definition and target group** — Terraform won't revert changes made by deployments.

### During deployments (managed by ecs-gitflow)

- CodeDeploy creates a new task set with an updated task definition (new image)
- Routes test listener (port 4443) to the green target group for validation
- After merge: shifts production listener (port 443) from blue to green
- Old task set drains and terminates (120 min wait for non-dev, 0 for dev)

### Datadog sidecar

When `create_datadog = true` (default), the module adds a Datadog agent container to the task:

- Image: `public.ecr.aws/datadog/agent:latest`
- Port 8126 (APM traces)
- Memory: 256 MB reserved
- API key from SSM: `/{account_id}/datadog/api-key`
- The app container memory is automatically reduced to fit within the task limit

---

## Module Interactions

### Inputs From

| Source | What It Provides |
| --- | --- |
| [config-manager](./config-manager.md) | `config.app.ecs` (container settings, security group, ECR URL, environment variables, secrets) |
| [config-manager](./config-manager.md) | `config.net.vpc_id`, `config.net.private_subnets` |
| [config-manager](./config-manager.md) | `config.app.shared_security_group`, `config.app.private_cert_arn` |
| ALB resources (inline) | `aws_alb_target_group.blue_fleet.id` for initial service registration |
| CloudWatch log group (inline) | Log group name for container logging |
| `app-env.json` (via config-manager) | Container environment, secrets, Datadog config |

### Outputs Consumed By

| Consumer | What It Uses |
| --- | --- |
| [ecs-gitflow](./ecs-gitflow.md) | `ecs_task_execution_role_arn` — IAM role for CodeDeploy to update task defs |
| [ecs-gitflow](./ecs-gitflow.md) | `ecs_service_name`, `ecs_cluster_name` — identifies the deployment target |
| CodeDeploy | Updates task definition with new image during deployment |

---

## Inputs

| Variable | Type | Required | Description |
| --- | --- | --- | --- |
| `config` | object | Yes | Config-manager output |
| `app_name` | string | No | Override app name (default: from config) |
| `aws_cloudwatch_log_group_name` | string | Yes | CloudWatch log group for app container |
| `aws_alb_target_group_arn` | string | No | Blue target group ARN. If set → CodeDeploy controller. If null → rolling updates. |
| `app_container_environment` | list(object) | No | Additional env vars for app container (merged with config defaults) |
| `app_container_secrets` | list(object) | No | Additional secrets for app container |
| `datadog_container_environment` | list(object) | No | Additional env vars for Datadog sidecar |
| `task_definition_cpu` | string | No | Task-level CPU (default from config) |
| `task_definition_memory` | string | No | Task-level memory (default from config) |
| `app_container_memory` | string | No | App container memory soft limit (auto-capped if Datadog enabled) |
| `create_datadog` | bool | No | Enable Datadog sidecar (default: true) |

---

## Sub-modules

| Sub-module | Purpose |
| --- | --- |
| `modules/autoscaling/alb_requests` | Step scaling based on ALB RequestCount metric |
| `modules/autoscaling/cpu` | Step scaling based on ECS CPUUtilization metric |

---

## Companion Resources (defined in service's `alb.tf`, not in this module)

The ECS module expects these to exist — they're defined inline in the service repo:

- `aws_alb` — Internal ALB with shared security group
- `aws_alb_target_group` × 2 — Blue (production) + Green (test), IP target type, health check on `/ping`
- `aws_alb_listener` (443) — Production HTTPS → blue target group
- `aws_alb_listener` (4443) — Test HTTPS → green target group

Both listeners have `lifecycle { ignore_changes = [default_action] }` because CodeDeploy swaps target groups during deployments.

---

## Usage Example

```hcl
module "ecs" {
  source  = "env1.toluna.me/toluna/terraform-aws-ecs/aws"
  version = "3.0.2"

  config                        = module.config
  app_name                      = local.app_name
  aws_cloudwatch_log_group_name = aws_cloudwatch_log_group.main.name
  aws_alb_target_group_arn      = aws_alb_target_group.blue_fleet.id
  task_definition_cpu           = "256"
  task_definition_memory        = "512"
  app_container_memory          = "256"

  app_container_environment = concat(
    module.config.app.ecs.app_container_environment,
    [{ name = "DD_RUNTIME_METRICS_ENABLED", value = "true" }]
  )

  datadog_container_environment = concat(
    module.config.app.ecs.datadog_container_environment,
    [{ name = "DD_APM_IGNORE_RESOURCES", value = "GET /ping" }]
  )
}
```
