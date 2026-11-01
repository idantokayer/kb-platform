---
title: Deployment Flow
description: The complete deployment lifecycle for Lambda, ECS, and Frontend services — from PR creation through CI/CD stages, traffic switch, and blue/green mechanics per service type.
tags: [plm, ci-cd, deployment, blue-green, lambda, ecs, frontend]
---

# Deployment Flow

The complete deployment lifecycle for all service types, from PR creation through production traffic serving.

Part of [Pipelines](./index.md). Orchestrated by [pipeline-controller](../terraform-modules/pipeline-controller.md); traffic is flipped by [switch-traffic](../terraform-modules/switch-traffic.md).

---

## Overview

All service types share the same outer flow managed by `pipeline-controller`, but differ in their inner deployment mechanism:

| Phase | Lambda | ECS | Frontend |
| --- | --- | --- | --- |
| PR Build | `bg_controller` CodeBuild | Same | Same |
| CI Build | [lambda-gitflow](../terraform-modules/lambda-gitflow.md) build stage | [ecs-gitflow](../terraform-modules/ecs-gitflow.md) build stage | [frontend-pipeline](../terraform-modules/frontend-pipeline.md) build stage |
| Deploy | CodeDeploy Lambda alias shift | CodeDeploy ECS blue/green | S3 sync + CloudFront invalidation |
| Traffic Switch | `switch-traffic` DNS flip (merge) | `switch-traffic` DNS flip (merge) | `switch-traffic` SPA DNS (merge) |
| Rollback | CodeDeploy auto-rollback | CodeDeploy auto-rollback | S3 versioning restore |

---

## Phase 1: PR Opened → PR Build

**Module:** `pipeline-controller` → `bg_controller` sub-module
**Trigger:** Bitbucket webhook `PULL_REQUEST_CREATED` or `PULL_REQUEST_UPDATED`
**CodeBuild project:** `codebuild-source-pr-{app_name}-{env_name}`

```
Developer opens PR on Bitbucket
    │
    ▼
Bitbucket webhook fires
    │
    ▼
bg_controller CodeBuild project starts
    │
    ├── 1. Concurrent PR guard
    │      Read SSM pr_id + check Bitbucket API
    │      If another PR is open → DECLINE this PR
    │
    ├── 2. In-progress pipeline guard
    │      If CodePipeline is running → DECLINE this PR
    │
    ├── 3. Stale pipeline cleanup
    │      Compare commit IDs
    │      If new commit on same PR → stop old pipeline + CodeDeploy deployments
    │
    ├── 4. Branch sync
    │      If PR branch is behind base → auto-sync
    │
    ├── 5. Infrastructure apply (blue/green workspace)
    │      terraform init
    │      terraform workspace select {env}-blue (or green)
    │      terraform apply -auto-approve
    │      Creates: Lambda functions / ECS service / CloudFront in the new color
    │
    ├── 6. Write metadata to SSM + S3
    │      SSM: commit_id, pr_id
    │      S3: ci.txt, pr.txt, src_changed.txt, commit_id.txt, head.txt
    │
    └── 7. Upload source artifact
           s3://{s3-codepipeline-{app}-{env_type}}/{env_name}/source_artifacts.zip
```

**Key detail:** The PR build creates infrastructure in a blue/green workspace (e.g., `qa-blue` or `qa-green`). This is the "green" environment that will receive the new code.

---

## Phase 2: Pipeline Triggered

**Module:** `pipeline-controller` → `pipeline_trigger` sub-module

```
S3 artifact uploaded (source_artifacts.zip)
    │
    ▼
S3 event notification → pipeline_trigger Lambda
    │
    ▼
Lambda calls codepipeline.startPipelineExecution()
    │
    ▼
CodePipeline starts (5 stages)
```

The pipeline itself is created by the gitflow module (`lambda-gitflow`, `ecs-gitflow`, or `frontend-pipeline`) in the app layer. The trigger mechanism is the same for all service types.

---

## Phase 3: CI/CD Pipeline Execution

### Stage 1: Source (All Types)

Downloads `source_artifacts.zip` from S3. Output: `source_output` artifact.

### Stage 2: CI Build

#### Lambda Services (`lambda-gitflow` → `build` sub-module)

```
CodeBuild: codebuild-build-{app}-{env}
    │
    ├── ECR login (app repo + base image repo)
    ├── Install datadog-ci CLI
    ├── Determine IMAGE_TAG
    │     CI: latest
    │     Dev: {env_name}
    ├── Calculate SemVer (bump minor from latest Bitbucket tag)
    ├── Report INPROGRESS to Bitbucket
    │
    ├── docker build (multi-stage)
    │     .NET: dotnet restore → test → publish
    │     Python: pip install → pytest → sonar-scanner
    │
    ├── docker push (tags: latest, {version}, {env_name})
    ├── Create Bitbucket tag (CI only)
    └── Report SUCCESSFUL to Bitbucket
```

Output: `ci_output` containing `new_version.txt`, `ci.txt`, `pr.txt`, `src_changed.txt`, `commit_id.txt`, `image_tag.txt`

#### ECS Services (`ecs-gitflow` → `build` sub-module)

Identical flow to Lambda. Same buildspec template structure, same ECR push, same SemVer tagging. The Docker image is the same — the difference is in how it's deployed (ECS task definition vs Lambda function).

#### Frontend Services (`frontend-pipeline` → `build` sub-module)

```
CodeBuild: codebuild-build-{app}-{env}
    │
    ├── Install Node.js 20
    ├── Configure npm auth (@toluna-ui-toolkit from Azure DevOps)
    ├── Calculate SemVer
    ├── Report INPROGRESS to Bitbucket
    │
    ├── npm install
    ├── npm run build
    ├── Copy env config: configs/envConfigs.{ENV_NAME}.js → build/configs/envConfigs.js
    ├── npm test (unit tests)
    ├── SonarQube scan + quality gate
    ├── Create Bitbucket tag
    └── Report SUCCESSFUL to Bitbucket
```

Output: `build/**` static files as pipeline artifact. No Docker image, no ECR.

### Stage 3: Pre-Deploy

#### Lambda (`lambda-gitflow` → `pre` sub-module)

One CodeBuild action per Lambda function, running in parallel:

```
CodeBuild: codebuild-pre-{app}-{env} (per function)
    │
    ├── Read ci_output artifacts
    ├── Resolve target image tag
    │     CI + src changed: copy ECR image from {from_env} → {env}-{commit}-{pr}
    │     CI + no src change: use existing {env} tag
    │     Dev: use new_version.txt
    │
    ├── Update Lambda function code
    │     aws lambda update-function-code --image-uri {new_image}
    │     aws lambda wait function-updated
    │
    ├── Publish new version
    │     aws lambda publish-version → TARGET_VERSION
    │     aws lambda wait published-version-active
    │
    ├── Get current live version
    │     aws lambda get-alias --name live → CURRENT_VERSION
    │
    └── Generate appspec.json
          { CurrentVersion: N, TargetVersion: N+1 }
```

Output: `cd_{function_name}_output` containing `appspec.json`

#### ECS (`ecs-gitflow` → `pre` sub-module)

```
CodeBuild: codebuild-pre-{app}-{env}
    │
    ├── Read ci_output artifacts
    ├── Resolve target image tag (same logic as Lambda)
    │
    ├── Generate imageDetail.json
    │     { "ImageURI": "{ecr_repo}:{target_tag}" }
    │
    ├── Generate taskdef.json
    │     Fetch current task def → update image + DD_VERSION
    │     Replace image with <IMAGE1_NAME> placeholder
    │
    └── Generate appspec.json
          { ContainerName, ContainerPort, hooks }
```

Output: `cd_output` containing `imageDetail.json`, `taskdef.json`, `appspec.json`

#### Frontend

No pre-deploy stage. Build output goes directly to deployment.

### Stage 4: Deploy

#### Lambda — CodeDeploy Alias Shift

```
CodeDeploy application: lambda-deploy-{app}-{env}
Deployment group: lambda-deploy-group-{function}-{env} (one per function)
Strategy: CodeDeployDefault.LambdaAllAtOnce

    ┌─────────────────────────────────────────────┐
    │  For each Lambda function (parallel):        │
    │                                              │
    │  1. CodeDeploy reads appspec.json            │
    │  2. BeforeAllowTraffic hook fires            │
    │     → Invokes merge_waiter Lambda            │
    │       (or test-framework-manager)            │
    │  3. Hook Lambda:                             │
    │     - Counts InProgress deployments          │
    │     - Stores in DynamoDB MergeWaiter table   │
    │     - When ALL functions report in:           │
    │       Posts "READY FOR MERGE" to Bitbucket   │
    │     - Does NOT call putLifecycleHookStatus   │
    │       (waits for PR merge)                   │
    │  4. Deployment PAUSES here                   │
    │     (waiting for hook status callback)       │
    └─────────────────────────────────────────────┘
```

The `live` alias still points to the old version. Traffic continues flowing to the old code. The deployment is paused, waiting for the merge step to call back.

#### ECS — CodeDeploy Blue/Green

```
CodeDeploy application: ecs-deploy-{app}-{env}
Deployment group: ecs-deploy-group-{app}-{env}
Strategy: CodeDeployDefault.ECSAllAtOnce

    ┌─────────────────────────────────────────────┐
    │  1. CodeDeploy reads taskdef.json            │
    │     Replaces <IMAGE1_NAME> with actual URI   │
    │  2. Creates new ECS task set (green)         │
    │     with updated task definition             │
    │  3. Routes TEST listener (4443) → green TG   │
    │  4. AfterAllowTestTraffic hook fires         │
    │     → test-framework-manager runs tests      │
    │       against port 4443                      │
    │  5. BeforeAllowTraffic hook fires            │
    │     → merge_waiter posts READY FOR MERGE     │
    │     → Deployment PAUSES                      │
    │  6. Waiting for PR merge...                  │
    └─────────────────────────────────────────────┘
```

Both blue and green task sets are running. Production listener (443) still routes to blue. Test listener (4443) routes to green for validation.

#### Frontend — S3 + CloudFront

```
    ┌─────────────────────────────────────────────┐
    │  1. Build artifacts → test S3 bucket         │
    │  2. Invalidate test CloudFront cache         │
    │  3. Test stage: Cypress E2E against test URL │
    │  4. MANUAL APPROVAL gate                     │
    │     Reviewer validates on test URL            │
    │  5. Artifacts → live S3 bucket               │
    │  6. Invalidate live CloudFront cache         │
    └─────────────────────────────────────────────┘
```

No CodeDeploy. The "blue/green" is test CloudFront vs live CloudFront.

### Stage 5: Post-Deploy

#### Lambda (`lambda-gitflow` → `post` sub-module)

```
CodeBuild: codebuild-post-{app}-{env}
    │
    ├── ECR image re-tagging (promotion)
    │     CD + src changed: copy {from_env} → {env_name}
    │     CI + src changed: copy {version} → {env}-{commit}-{pr}
    ├── Post SUCCESSFUL to Bitbucket commit status
    ├── Jira release automation (prod only)
    └── Re-enable Lambda warmers (EventBridge schedules)
```

#### ECS (`ecs-gitflow` → `post` sub-module)

```
CodeBuild: codebuild-post-{app}-{env}
    │
    ├── ECR image re-tagging (same as Lambda)
    ├── Post SUCCESSFUL to Bitbucket commit status
    ├── Jira release automation (prod only)
    └── Subscribe Datadog log forwarder to new log group
          Reads current_color from Consul
          Subscribes {app}-{env}-{color} log group
```

#### Frontend

Post-deploy handled inline — Bitbucket status update and notification.

---

## Phase 4: PR Merge → Traffic Switch

**Module:** `pipeline-controller` → `bg_controller` (merge buildspec)
**Trigger:** Bitbucket webhook `PULL_REQUEST_MERGED`
**CodeBuild project:** `codebuild-source-merge-{app_name}-{env_name}`

This is the critical step that flips live traffic to the new deployment.

### Lambda Merge Flow

```
Merge buildspec starts
    │
    ├── 1. Read DynamoDB MergeWaiter table
    │      Get all {DeploymentId, LifecycleEventHookExecutionId} entries
    │
    ├── 2. Call back ALL CodeDeploy hooks
    │      For each entry:
    │        aws deploy put-lifecycle-event-hook-execution-status \
    │          --deployment-id {id} \
    │          --lifecycle-event-hook-execution-id {hookId} \
    │          --status Succeeded
    │      This unblocks all paused deployments
    │      CodeDeploy shifts each Lambda's `live` alias → new version
    │
    ├── 3. Run switch-traffic
    │      terraform apply -target=module.switch-traffic
    │      This updates:
    │        - DNS records (Route53 → API Gateway / VPC endpoint)
    │        - Lambda event source mappings (SQS → new `live` alias)
    │        - EventBridge schedules (→ new `live` alias)
    │        - SNS subscriptions
    │        - Step Function `live` aliases
    │
    ├── 4. Wait 300 seconds (traffic drain)
    │
    └── 5. Destroy old environment
           terraform workspace select {env}-{old_color}
           terraform destroy -auto-approve
           Removes old Lambda versions, old API Gateway stage, etc.
```

### ECS Merge Flow

```
Merge buildspec starts
    │
    ├── 1. Call back CodeDeploy hook (single deployment)
    │      aws deploy put-lifecycle-event-hook-execution-status --status Succeeded
    │      CodeDeploy shifts production listener (443) → green target group
    │
    ├── 2. Run switch-traffic
    │      terraform apply -target=module.switch-traffic
    │      Updates DNS records (Route53 → ALB)
    │
    ├── 3. Wait 300 seconds
    │
    └── 4. Destroy old environment
           Old blue task set terminated
           (CodeDeploy handles blue instance termination after wait period)
```

### Frontend Merge Flow

```
Merge buildspec starts
    │
    ├── 1. Run switch-traffic
    │      terraform apply -target=module.switch-traffic
    │      Updates DNS (Route53 → CloudFront distribution)
    │
    └── 2. No destroy needed
           (S3 versioning preserves old content for rollback)
```

---

## Blue/Green Mechanism by Service Type

### Lambda Blue/Green

```
BEFORE DEPLOY:
  Lambda function: my-func-qa
  Alias "live" → Version 5
  API Gateway integration → my-func-qa:live
  SQS event source → my-func-qa:live

DURING DEPLOY (paused):
  Lambda function: my-func-qa
  Version 6 published (new code)
  Alias "live" → still Version 5 (CodeDeploy hasn't shifted yet)
  All traffic still goes to Version 5

AFTER MERGE (CodeDeploy completes):
  Alias "live" → Version 6
  All integrations automatically route to Version 6
  (because they reference :live, not a specific version)

AFTER DESTROY:
  Old version 5 still exists but unreferenced
  Old blue/green workspace resources destroyed
```

The `live` alias is the key abstraction. All integrations (API Gateway, SQS, SNS, EventBridge, Step Functions) reference `{function_arn}:live`. CodeDeploy shifts what `live` points to.

### ECS Blue/Green

```
BEFORE DEPLOY:
  ECS Service: my-svc-qa
  Task Set: blue (running Version A)
  ALB Listener 443 → Blue Target Group → Blue Tasks
  ALB Listener 4443 → Green Target Group → (empty)

DURING DEPLOY (paused):
  Task Set: blue (Version A) — still serving production
  Task Set: green (Version B) — created by CodeDeploy
  ALB Listener 443 → Blue Target Group → Blue Tasks (production)
  ALB Listener 4443 → Green Target Group → Green Tasks (test)

AFTER MERGE (CodeDeploy completes):
  ALB Listener 443 → Green Target Group → Green Tasks (now production)
  Blue tasks draining (termination wait: 120 min non-dev, 0 dev)

AFTER TERMINATION WAIT:
  Blue task set terminated
  Green is now the only running task set
```

Two ALB target groups (blue/green) and two listeners (443 prod, 4443 test) enable zero-downtime switching.

### Frontend Blue/Green

```
BEFORE DEPLOY:
  Live S3 bucket → CloudFront (live) → production domain
  Test S3 bucket → CloudFront (test) → test domain

DURING DEPLOY:
  New build artifacts → Test S3 bucket
  Test CloudFront invalidated → serves new content
  Live S3 bucket unchanged → production still serves old content

AFTER APPROVAL + DEPLOY:
  New build artifacts → Live S3 bucket
  Live CloudFront invalidated → serves new content
  (Brief cache propagation window, typically <60s)

ROLLBACK:
  S3 versioning → restore previous object versions
  CloudFront invalidation → serve restored content
```

No true zero-downtime — CloudFront edge cache propagation creates a brief window of mixed content.

---

## CD (Continuous Deployment) Pipeline Variant

For production environments, the pipeline type is `cd` instead of `ci`:

```
CI pipeline (non-prod):
  Build → Push image → Deploy → Test → READY FOR MERGE

CD pipeline (prod):
  No build — promotes existing image from non-prod
  Pre-Deploy: copies ECR image tag from {from_env} (e.g., "qa") to {env_name} (e.g., "prod")
  Deploy: same CodeDeploy flow
  merge_waiter: triggers merge build directly (no PR merge needed)
```

The `from_env` config in `app-env.json` controls which environment's image gets promoted. For example, prod's `from_env = "qa"` means the QA-tested image is promoted to production. See [Pipelines Types](./pipelines-types.md).

---

## Dev Pipeline Variant

Developer overlay environments use `pipeline_type = "dev"`:

```
Differences from CI:
  - No SemVer tagging (uses env_name as image tag)
  - No Bitbucket tag creation
  - switch-traffic runs in APP layer (not shared)
  - SNS/EventBridge integrations created in APP layer (not shared)
  - merge_waiter uses qa-merge-waiter (not env-specific)
  - CodeDeploy termination wait = 0 (immediate)
```

---

## Pipeline State Diagram

```
PR_OPENED ──► PR_BUILD ──► ARTIFACT_UPLOADED ──► PIPELINE_STARTED
                                                       │
                                          ┌────────────┼────────────┐
                                          ▼            ▼            ▼
                                       SOURCE        BUILD      PRE_DEPLOY
                                                                    │
                                                                    ▼
                                                                 DEPLOY
                                                                    │
                                                              (hooks fire)
                                                                    │
                                                                    ▼
                                                          READY_FOR_MERGE
                                                                    │
                                                              (PR merged)
                                                                    │
                                                                    ▼
                                                              MERGE_BUILD
                                                                    │
                                                    ┌───────────────┼──────────┐
                                                    ▼               ▼          ▼
                                              HOOK_CALLBACK   SWITCH_TRAFFIC  DESTROY_OLD
                                                                    │
                                                                    ▼
                                                              POST_DEPLOY
                                                                    │
                                                                    ▼
                                                                 DONE
```
