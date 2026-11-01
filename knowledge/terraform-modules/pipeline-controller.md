---
title: pipeline-controller
description: Manages the complete CI/CD orchestration layer — Bitbucket webhooks, PR/merge CodeBuild projects, pipeline triggering, deployment coordination, rollback, and notifications.
tags: [plm, terraform, module, pipeline-controller, ci-cd, shared, blue-green]
---

# pipeline-controller

Manages the complete CI/CD orchestration layer: Bitbucket webhooks, PR/merge CodeBuild projects, pipeline triggering, deployment coordination, rollback, and notifications.

Part of [Terraform Modules](./index.md). Release notes: [3.1.7](./pipeline-controller-3.1.7.md). See the [Deployment Flow](../pipelines/deployment-flow.md) for how this module orchestrates deployments end-to-end.

---

## Quick Reference

| Property | Value |
| --- | --- |
| **Registry** | `env1.toluna.me/toluna/terraform-aws-pipeline-controller/aws` |
| **Current Version** | 3.1.7 |
| **Repository** | [tolunaengineering/terraform-aws-pipeline-controller](https://bitbucket.org/tolunaengineering/terraform-aws-pipeline-controller) |
| **Present In** | Shared layer only |

---

## Purpose

The pipeline-controller is the **entry point for all deployments**. It receives Bitbucket webhook events and orchestrates the entire deployment lifecycle — from PR validation through merge and traffic switching. Without this module, no code can be deployed.

---

## Runtime Behavior (what happens during a deployment)

### PR Opened/Updated

1. Bitbucket fires webhook → `codebuild-source-pr` starts
2. PR build checks for concurrent PRs (declines if another is open)
3. Checks if a pipeline is already running (declines if so)
4. If new commit on existing PR: stops old pipeline + old CodeDeploy deployments
5. Syncs branch if behind base
6. Runs `terraform apply` on blue/green workspace (creates new Lambda/ECS infra)
7. Writes metadata to SSM and S3 (`ci.txt`, `pr.txt`, `commit_id.txt`, etc.)
8. Uploads `source_artifacts.zip` to S3 artifact bucket

### Artifact Uploaded

9. S3 notification triggers `pipeline_trigger` Lambda
10. Lambda calls `codepipeline.startPipelineExecution()`

### During CodeDeploy

11. CodeDeploy invokes `merge_waiter` Lambda as `BeforeAllowTraffic` hook
12. Lambda counts in-progress deployments, stores in DynamoDB
13. When ALL functions report in: posts "READY FOR MERGE" comment to Bitbucket PR
14. Does NOT call `putLifecycleHookStatus` — deployment **pauses here**

### PR Merged

15. Bitbucket fires merge webhook → `codebuild-source-merge` starts
16. Reads DynamoDB MergeWaiter table for all deployment/hook IDs
17. Calls `putLifecycleHookStatus(Succeeded)` for each → CodeDeploy completes alias shift
18. Runs `terraform apply -target=module.switch-traffic` (flips DNS, event sources)
19. Waits 300 seconds (traffic drain)
20. Runs `terraform destroy` on old blue/green workspace

### PR Declined/Closed (new in 3.1.7)

21. Bitbucket fires `PULL_REQUEST_CLOSED` webhook → `codebuild-source-pr` starts
22. Buildspec detects PR state is DECLINED or SUPERSEDED in `pre_build`
23. Calls `cleanup_and_exit()` — stops running pipeline, stops CodeDeploy deployments
24. Cleans up DynamoDB entries and releases environment ownership
25. Build exits cleanly — environment is immediately available for other PRs

### On Failure

26. `rollback_controller` Lambda detects failed deployment
27. Stops CodeDeploy, reverts alias if partially shifted
28. `notifier` Lambda publishes failure to SNS

---

## Module Interactions

### Inputs From

| Source | What It Provides |
| --- | --- |
| [config-manager](./config-manager.md) | `config.shared.controller` — environment list, pipeline branches, webhook config |
| [config-manager](./config-manager.md) | `config.globals` — app_name, tribe, env_type, profiles |

### Outputs Consumed By

| Consumer | What It Uses |
| --- | --- |
| [lambda-gitflow](./lambda-gitflow.md) / [ecs-gitflow](./ecs-gitflow.md) | The CodePipeline these modules create is **started by** pipeline_trigger Lambda |
| [lambda-gitflow](./lambda-gitflow.md) CodeDeploy | `BeforeAllowTraffic` hook invokes merge_waiter Lambda |
| Merge buildspec | Calls [switch-traffic](./switch-traffic.md) module via `terraform apply -target` |
| S3 artifact bucket | Shared with gitflow modules for artifact storage |

---

## Inputs

| Variable | Type | Required | Description |
| --- | --- | --- | --- |
| `config` | object | Yes | Config-manager output |
| `source_repository` | string | No | Bitbucket repo override (e.g., `tolunaengineering/machine-learning-boyaca-api`). Only needed when the repo name differs from the service name. If omitted, config-manager injects the service name as the repository slug. |
| `apps` | map(object) | No | Multi-app config override |
| `vpc_config` | map(object) | No | VPC configuration for CodeBuild |

---

## Sub-modules

| Sub-module | Purpose |
| --- | --- |
| `bg_controller` | Blue/green CodeBuild projects (PR + merge). The "brains" of deployment orchestration. |
| `pipeline_trigger` | Lambda: S3 event → starts CodePipeline |
| `cd_listener` | Lambda: listens for CD pipeline completion, triggers prod deployment |
| `merge_waiter` | Lambda: CodeDeploy hook coordinator, DynamoDB tracking, Bitbucket commenting |
| `finalize_controller` | Lambda: post-merge cleanup |
| `rollback_controller` | Lambda: deployment failure recovery |
| `notifier` | Lambda: SNS notifications on pipeline state changes |

---

## Usage Example

```hcl
# Standard — repo name matches service name (no source_repository needed)
module "source_pipeline_controller" {
  source  = "env1.toluna.me/toluna/terraform-aws-pipeline-controller/aws"
  version = "3.1.7"

  config = module.config
}

# Override — when repo name differs from service name
module "source_pipeline_controller" {
  source  = "env1.toluna.me/toluna/terraform-aws-pipeline-controller/aws"
  version = "3.1.7"

  config            = module.config
  source_repository = "tolunaengineering/machine-learning-boyaca-api"
}
```
