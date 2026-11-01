---
title: lambda-gitflow
description: Creates the full CI/CD pipeline for Lambda services — CodeBuild (build, pre-deploy, post-deploy), CodePipeline, and CodeDeploy for blue/green Lambda alias shifting.
tags: [plm, terraform, module, lambda-gitflow, ci-cd, app-layer]
---

# lambda-gitflow

Creates the full CI/CD pipeline for Lambda services: CodeBuild (build, pre-deploy, post-deploy), CodePipeline, and CodeDeploy for blue/green Lambda alias shifting.

Part of [Terraform Modules](./index.md). Works with [pipeline-controller](./pipeline-controller.md) (which triggers it) and the [lambda](./lambda.md) module (which provides the function list for CodeDeploy). See the [Deployment Flow](../pipelines/deployment-flow.md).

---

## Quick Reference

| Property | Value |
| --- | --- |
| **Registry** | `env1.toluna.me/toluna/terraform-aws-lambda-gitflow/aws` |
| **Current Version** | 3.3.0 |
| **Repository** | [tolunaengineering/terraform-aws-lambda-gitflow](https://bitbucket.org/tolunaengineering/terraform-aws-lambda-gitflow) |
| **Present In** | App layer |

---

## Purpose

Lambda-gitflow creates the pipeline that builds, tests, and deploys Lambda functions using blue/green deployment via CodeDeploy. It works with the `pipeline-controller` (which triggers it) and the `lambda` module (which provides the function list for CodeDeploy).

---

## Runtime Behavior

### At terraform apply time

- Creates CodeBuild projects (build, pre-deploy per function, post-deploy)
- Creates CodePipeline with 5 stages
- Creates CodeDeploy application + one deployment group per Lambda function
- All deployment groups use `LambdaAllAtOnce` strategy with `BeforeAllowTraffic` lifecycle hook

### During a deployment (runtime)

**Build stage:**

1. ECR login (app repo + base image repo)
2. Docker build (multi-stage: restore → test → publish)
3. Docker push to ECR with tags: `latest`, `{version}`, `{env_name}`
4. SemVer: bump minor from latest Bitbucket tag
5. Report build status to Bitbucket

**Pre-deploy stage (runs per function, in parallel):**

1. Resolve target image tag (CI: copy from `from_env`; Dev: use `new_version.txt`)
2. `aws lambda update-function-code --image-uri {new_image}`
3. `aws lambda publish-version` → new version number
4. Generate `appspec.json` with `{ CurrentVersion: N, TargetVersion: N+1 }`

**Deploy stage (CodeDeploy):**

1. Reads `appspec.json` per function
2. Fires `BeforeAllowTraffic` hook → invokes `merge_waiter` Lambda (from pipeline-controller)
3. Deployment **pauses** — waiting for PR merge to call back

**After PR merge (triggered by pipeline-controller merge build):**

1. `putLifecycleHookStatus(Succeeded)` unblocks CodeDeploy
2. CodeDeploy shifts `live` alias to new version
3. All integrations automatically route to new code

**Post-deploy stage:**

1. ECR image re-tagging (promotion between environments)
2. Bitbucket commit status update
3. Jira release automation (prod only)
4. Re-enable Lambda warmers

---

## Module Interactions

### Inputs From

| Source | What It Provides |
| --- | --- |
| [config-manager](./config-manager.md) | `config.app.pipeline_type` (dev/ci/cd), `config.app.ecr_repo_url`, `config.app.pipeline_branch` |
| [lambda](./lambda.md) module | `aws_lambda_out` for each function (function ARNs, names for CodeDeploy) |
| [pipeline-controller](./pipeline-controller.md) (indirect) | `pipeline_trigger` Lambda starts this CodePipeline; S3 artifact bucket stores source |

### Outputs Consumed By

| Consumer | What It Uses |
| --- | --- |
| [pipeline-controller](./pipeline-controller.md) merge_waiter | CodeDeploy invokes it as `BeforeAllowTraffic` hook |
| [pipeline-controller](./pipeline-controller.md) merge build | Runs [switch-traffic](./switch-traffic.md) after deployment completes |

---

## Inputs

| Variable | Type | Required | Description |
| --- | --- | --- | --- |
| `config` | object | Yes | Config-manager output |
| `function_list` | list | Yes | Lambda module outputs (`aws_lambda_out`) — one per function to deploy |
| `dockerfile_path` | string | Yes | Docker build context path (e.g., `"service"`) |
| `source_repository` | string | No | Bitbucket repo override (only when repo name ≠ service name) |
| `termination_wait_time_in_minutes` | number | No | Old version drain time (default: 120, auto 0 for dev) |
| `enable_jira_automation` | bool | No | Jira release automation on prod deploy |
| `run_integration_tests` | bool | No | Run Postman tests during deploy |
| `run_stress_tests` | bool | No | Run stress tests (default: false) |

---

## Sub-modules

| Sub-module | Purpose |
| --- | --- |
| `build` | CodeBuild project: Docker build, ECR push, SemVer tagging, Bitbucket status |
| `pre` | CodeBuild project(s) — one per Lambda. Updates function code, publishes version, generates `appspec.json` |
| `code-deploy` | CodeDeploy application + deployment groups. `LambdaAllAtOnce` + `BeforeAllowTraffic` hook |
| `ci-cd-code-pipeline` | CodePipeline: Source → Build → Pre → Deploy → Post |
| `post` | CodeBuild project: ECR re-tagging, Bitbucket status, Jira automation, warmer re-enable |

---

## Usage Example

```hcl
module "pipeline" {
  source  = "env1.toluna.me/toluna/terraform-aws-lambda-gitflow/aws"
  version = "3.3.0"

  config          = module.config
  dockerfile_path = "service"
  function_list   = [
    module.lambda_sqs_listener.aws_lambda_out,
    module.lambda_sqs_api.aws_lambda_out,
    module.lambda_health.aws_lambda_out,
    module.lambda_dynamo.aws_lambda_out
  ]
}
```
