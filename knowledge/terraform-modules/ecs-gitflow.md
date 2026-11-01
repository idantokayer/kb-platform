---
title: ecs-gitflow
description: Creates the full CI/CD pipeline for ECS Fargate services — CodePipeline, CodeBuild (build, pre, post), and CodeDeploy for blue/green via ALB target group switching.
tags: [plm, terraform, module, ecs-gitflow, ci-cd, app-layer]
---

# ecs-gitflow

Creates a complete CI/CD pipeline for ECS Fargate services: CodePipeline, CodeBuild (build, pre-deploy, post-deploy), and CodeDeploy for blue/green deployment via ALB target group switching.

Part of [Terraform Modules](./index.md). Works with [pipeline-controller](./pipeline-controller.md) (which triggers it) and the [ecs](./ecs.md) module (service/cluster/role). See the [Deployment Flow](../pipelines/deployment-flow.md).

---

## Quick Reference

| Property | Value |
| --- | --- |
| **Registry** | `env1.toluna.me/toluna/terraform-aws-ecs-gitflow/aws` |
| **Current Version** | 3.3.0 |
| **Repository** | [tolunaengineering/terraform-aws-ecs-gitflow](https://bitbucket.org/tolunaengineering/terraform-aws-ecs-gitflow) |
| **Present In** | App layer |

---

## Purpose

ECS-gitflow creates the pipeline that builds, tests, and deploys ECS Fargate services using blue/green deployment via CodeDeploy with ALB traffic shifting. Structurally similar to [lambda-gitflow](./lambda-gitflow.md) but targets ECS services instead of Lambda functions.

It works with `pipeline-controller` (which triggers it), the `ecs` module (which provides the service/cluster/role), and the ALB resources (which provide blue/green target groups and listeners).

---

## Runtime Behavior

### At terraform apply time

- Creates 3 CodeBuild projects (build, pre-deploy, post-deploy)
- Creates CodePipeline with 5 stages
- Creates CodeDeploy application + deployment group with blue/green config
- Deployment group references both ALB target groups and both listeners (prod 443, test 4443)
- Strategy: `CodeDeployDefault.ECSAllAtOnce` with auto-rollback on failure

### During a deployment (runtime)

**Build stage:**

1. ECR login (app repo + base image repo)
2. Docker build (multi-stage: install → test → sonar → runtime image)
3. Docker push to ECR with tags: `latest`, `{version}`, `{env_name}`
4. SemVer bump from latest Bitbucket tag
5. Report build status to Bitbucket

**Pre-deploy stage:**

1. Resolve target image tag (CI: copy from `from_env`; Dev: use `new_version.txt`)
2. Generate `imageDetail.json` with the target image URI
3. Fetch current task definition via `aws ecs describe-task-definition`
4. Generate updated `taskdef.json` — replaces image with `<IMAGE1_NAME>` placeholder, updates `DD_VERSION`
5. Generate `appspec.json` with container name, port, and lifecycle hooks

**Deploy stage (CodeDeploy):**

1. CodeDeploy reads `taskdef.json`, replaces `<IMAGE1_NAME>` with actual URI from `imageDetail.json`
2. Creates new ECS task set (green) with updated task definition
3. Routes test listener (4443) → green target group
4. `AfterAllowTestTraffic` hook → invokes `test-framework-manager` Lambda (runs integration tests against port 4443)
5. `BeforeAllowTraffic` hook → invokes `merge_waiter` Lambda (posts "READY FOR MERGE" to Bitbucket)
6. Deployment **pauses** — waiting for PR merge

**After PR merge (triggered by pipeline-controller merge build):**

1. `putLifecycleHookStatus(Succeeded)` unblocks CodeDeploy
2. CodeDeploy shifts production listener (443) → green target group
3. Blue task set enters termination wait (120 min non-dev, 0 dev)
4. After wait: blue tasks terminated

**Post-deploy stage:**

1. ECR image re-tagging (promotion between environments)
2. Bitbucket commit status update
3. Jira release automation (prod only)
4. Datadog log subscription — subscribes the new log group (`{app}-{env}-{color}`) to Datadog forwarder

---

## Module Interactions

### Inputs From

| Source | What It Provides |
| --- | --- |
| [config-manager](./config-manager.md) | `config.app.ecs_pipeline` (pipeline type, branch, from_env, ECR URL) |
| [ecs](./ecs.md) module | `ecs_task_execution_role_arn` — IAM role for CodeDeploy |
| ALB resources (inline) | `alb_listener_arn` (443), `alb_test_listener_arn` (4443), `alb_tg_blue_name`, `alb_tg_green_name` |
| [pipeline-controller](./pipeline-controller.md) (indirect) | `pipeline_trigger` Lambda starts this CodePipeline; S3 artifact bucket stores source |

### Outputs Consumed By

| Consumer | What It Uses |
| --- | --- |
| [pipeline-controller](./pipeline-controller.md) merge_waiter | CodeDeploy invokes it as `BeforeAllowTraffic` hook |
| [test-framework](./test-framework.md) | CodeDeploy invokes `test-framework-manager` as `AfterAllowTestTraffic` hook |
| [pipeline-controller](./pipeline-controller.md) merge build | Runs [switch-traffic](./switch-traffic.md) after deployment completes |

---

## Inputs

| Variable | Type | Required | Description |
| --- | --- | --- | --- |
| `config` | object | Yes | Config-manager output |
| `alb_listener_arn` | string | Yes | Production ALB listener ARN (port 443) |
| `alb_test_listener_arn` | string | Yes | Test ALB listener ARN (port 4443) |
| `alb_tg_blue_name` | string | Yes | Blue target group name |
| `alb_tg_green_name` | string | Yes | Green target group name |
| `ecs_iam_roles_arns` | list(string) | Yes | Task execution role ARNs (from ecs module) |
| `source_repository` | string | No | Bitbucket repo override (only when repo name ≠ service name) |
| `termination_wait_time_in_minutes` | number | No | Old task set drain time (default: 120, auto 0 for dev) |
| `enable_jira_automation` | bool | No | Jira release automation on prod deploy |
| `run_integration_tests` | bool | No | Run Postman tests via test-framework during deploy |

---

## Sub-modules

| Sub-module | Purpose |
| --- | --- |
| `build` | CodeBuild project: Docker build, ECR push, SemVer tagging, Bitbucket status |
| `pre` | CodeBuild project: resolve image tag, generate taskdef.json + appspec.json + imageDetail.json |
| `code-deploy` | CodeDeploy application + blue/green deployment group with ALB traffic control |
| `ci-cd-code-pipeline` | CodePipeline: Source → Build → Pre → Deploy → Post |
| `post` | CodeBuild project: ECR re-tagging, Bitbucket status, Jira, Datadog log subscription |

---

## Key Difference from lambda-gitflow

| Aspect | lambda-gitflow | ecs-gitflow |
| --- | --- | --- |
| Deploy mechanism | CodeDeploy Lambda alias shift | CodeDeploy ECS blue/green with ALB listener swap |
| Deploy artifacts | `appspec.json` per function | `taskdef.json` + `appspec.json` + `imageDetail.json` |
| Traffic routing | Lambda `live` alias | ALB target groups (blue/green) + listeners (443/4443) |
| Test validation | Via merge_waiter only | Dedicated test listener (4443) + `AfterAllowTestTraffic` hook |
| Pre-deploy | Updates function code + publishes version | Fetches task def + generates deployment artifacts |
| Post-deploy extra | Re-enable warmers | Subscribe Datadog log forwarder to new log group |
| Termination | Old Lambda version unreferenced | Old ECS task set drained + terminated after wait |

---

## Usage Example

```hcl
module "pipeline" {
  source  = "env1.toluna.me/toluna/terraform-aws-ecs-gitflow/aws"
  version = "3.3.0"

  config                = module.config
  alb_listener_arn      = aws_alb_listener.https.arn
  alb_test_listener_arn = aws_alb_listener.http_test.arn
  alb_tg_blue_name     = aws_alb_target_group.blue_fleet.name
  alb_tg_green_name    = aws_alb_target_group.green_fleet.name
  ecs_iam_roles_arns   = [module.ecs.ecs_task_execution_role_arn]

  depends_on = [module.ecs]
}
```

The `depends_on` ensures the ECS service exists before the pipeline is created (CodeDeploy needs an active service to target).
