---
title: frontend-pipeline
description: Complete CI/CD pipeline for frontend SPA apps — build with SonarQube, deploy to a test S3 bucket, run Cypress E2E, then promote to production with CloudFront invalidation.
tags: [plm, terraform, module, frontend-pipeline, ci-cd, spa, cloudfront, app-layer]
---

# frontend-pipeline

Creates a complete CI/CD pipeline for frontend SPA applications using AWS CodePipeline and CodeBuild. Handles building, deploying to a test S3 bucket, running Cypress integration tests, and promoting to production with CloudFront cache invalidation.

Part of [Terraform Modules](./index.md). Triggered by [pipeline-controller](./pipeline-controller.md). See the [Deployment Flow](../pipelines/deployment-flow.md) and the [Garnish — Frontend SPA Template](../platform-microservice-templates/garnish.md).

---

## Quick Reference

| Property | Value |
| --- | --- |
| **Registry** | `env1.toluna.me/{namespace}/terraform-aws-frontend-pipeline/aws` |
| **Current Version** | 3.1.0 |
| **Repository** | [tolunaengineering/terraform-aws-frontend-pipeline](https://bitbucket.org/tolunaengineering/terraform-aws-frontend-pipeline) |
| **Present In** | App layer (per-environment) |
| **App Type** | `spa` (set in config-manager) |

---

## Purpose

Frontend SPA applications (React, Angular, etc.) hosted on S3 behind CloudFront need a different deployment model than backend services. Instead of Lambda/ECS blue-green deployments, frontends are built as static assets, deployed to a **test S3 bucket** for validation, tested with Cypress E2E tests, and then promoted to the **production S3 bucket** with CloudFront invalidation.

This module encapsulates the entire workflow:

- **Build** — Compiles the frontend application, runs SonarQube quality gate analysis, creates a SemVer tag in Bitbucket
- **Deploy to test** — Uploads built assets to a test S3 bucket for validation
- **Test** — Runs Cypress integration tests against the test deployment
- **Post-Deploy** — Syncs assets from test to production bucket and invalidates CloudFront caches

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CodePipeline Stages                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────┐    ┌───────┐ │
│  │  Source   │───▶│  Build   │───▶│     Deploy       │───▶│ Post  │ │
│  │  (S3)    │    │(CodeBuild│    │                  │    │Deploy │ │
│  │          │    │ +Sonar)  │    │ ┌──────────────┐ │    │       │ │
│  └──────────┘    └──────────┘    │ │Deploy to Test│ │    │ Sync  │ │
│                                  │ │   S3 Bucket  │ │    │Test→  │ │
│                                  │ └──────┬───────┘ │    │Prod   │ │
│                                  │        │         │    │       │ │
│                                  │ ┌──────▼───────┐ │    │ CF    │ │
│                                  │ │ Cypress Tests│ │    │Invali-│ │
│                                  │ │  (CodeBuild) │ │    │dation │ │
│                                  │ └──────┬───────┘ │    └───────┘ │
│                                  │        │         │              │
│                                  │ ┌──────▼───────┐ │              │
│                                  │ │Manual Approve│ │              │
│                                  │ │(non-dev only)│ │              │
│                                  │ └──────────────┘ │              │
│                                  └──────────────────┘              │
└─────────────────────────────────────────────────────────────────────┘

S3 Buckets:
  ┌─────────────┐        ┌─────────────┐        ┌─────────────┐
  │ src_bucket  │        │ test_bucket │        │target_bucket│
  │(prod bucket)│        │ (staging)   │        │(production) │
  └─────────────┘        └──────┬──────┘        └──────┬──────┘
                                │     Post-Deploy       │
                                └───── sync ───────────▶│
                                                        │
                                              ┌─────────▼─────────┐
                                              │   CloudFront      │
                                              │  (invalidated)    │
                                              └───────────────────┘
```

---

## Runtime Behavior

### Stage 1: Source

- CodePipeline polls S3 for `{env_name}/source_artifacts.zip` in the artifacts bucket (`s3-codepipeline-{app_name}-{env_type}`)
- Artifacts are uploaded by the [pipeline-controller](./pipeline-controller.md) (PR/merge build) before triggering this pipeline

### Stage 2: Build

- CodeBuild project `codebuild-build-{app_name}-{env_name}` runs the build
- Uses `buildspec.yml.tpl` template (must be provided at the Terraform root, NOT inside the module)
- Authenticates to Azure DevOps npm registry (`@toluna-ui-toolkit` scope) using ADO credentials from SSM
- Compiles the frontend app (`npm install` + `npm run build`)
- Copies environment-specific config: `envConfigs.{env_name}.js` → `build/configs/envConfigs.js`
- **For non-dev pipelines:** runs SonarQube quality gate analysis and reports result to Bitbucket. Creates a SemVer tag in Bitbucket (bumps minor version from latest tag)
- **For dev pipelines:** tags with the environment name instead of SemVer
- **If no source changes:** skips build entirely, syncs from `src_bucket` (production), applies env config, and invalidates test CloudFront
- Reports build status to Bitbucket commit via API (using `BB_USER`/`BB_PASS` basic auth)

### Stage 3: Deploy + Test + Approval

This stage has three ordered actions:

1. **Deploy (run_order: 1)** — Extracts build artifacts directly to the **test S3 bucket** with `public-read` ACL
2. **Test (run_order: 2)** — CodeBuild project `codebuild-test-{app_name}-{env_name}` runs Cypress integration tests using environment-specific config (`cypress.config.{env_name}.js`). Reports test status to Bitbucket via bearer token (`BB_TOKEN`). Supports Cypress Dashboard recording via `cypress_record_key`. Produces JUnit XML test reports.
3. **Merge Approval (run_order: 3, non-dev only)** — Manual approval gate labeled "Merge-Waiter". Only present when `pipeline_type != "dev"`. Blocks promotion until manually approved.

### Stage 4: Post-Deploy

CodeBuild project `codebuild-post-{app_name}-{env_name}` performs the production promotion:

- Syncs all files from **test bucket** to **production (target) bucket** via `aws s3 sync`
- Cleans the test bucket with `aws s3 rm --recursive`
- Invalidates **both** CloudFront distributions (production and test) with `/**/*` and `/*` paths
- Reports deployment success to Bitbucket commit status (using bearer token)
- If environment is `prod`: triggers a Jira release webhook

---

## Pipeline Types

| pipeline_type | Behavior |
| --- | --- |
| `dev` | No merge approval gate. Build output artifact named `dev_output`. No SonarQube. Tags with env name instead of SemVer. Fastest iteration cycle. |
| `ci` | Includes merge approval gate. Full SonarQube + SemVer tagging. Used for QA environments with PR-based deployments. |
| `cd` | Includes merge approval gate. Full SonarQube + SemVer tagging. Used for staging/production environments with continuous delivery. |

---

## Module Interactions

### Inputs From

| Source | What It Provides |
| --- | --- |
| [config-manager](./config-manager.md) | `config.app.frontend_pipeline` — app_name, env_type, env_name, from_env, pipeline_type, target_bucket, test_bucket, src_bucket, run_integration_tests, vpc_config |
| SSM Parameter Store | ADO credentials, Bitbucket auth, SonarQube config, Jira release hook, CodeStar connection (see Required SSM Parameters) |
| Caller (root module) | `distribution_id`, `test_distribution_id` (CloudFront), `security_group_ids` |

### Outputs Consumed By

| Consumer | What It Uses |
| --- | --- |
| Bitbucket PR | Commit status updates (build, test, deploy, SonarQube states) |
| Bitbucket Tags | SemVer version tags created on each CI/CD build |
| SonarQube | Code quality analysis results and quality gate status |
| CloudFront (production + test) | Cache invalidation after deployment |
| Jira (prod only) | Release webhook notification on successful prod deployment |

---

## Inputs

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `config` | any | Yes | — | Config-manager output. Must contain `config.app.frontend_pipeline` block. |
| `distribution_id` | string | Yes | — | Production CloudFront distribution ID for cache invalidation. |
| `test_distribution_id` | string | Yes | — | Test CloudFront distribution ID for cache invalidation. |
| `app_name` | string | No | From config | Application name. |
| `env_name` | string | No | From config | Environment name (e.g., qa, stg, prod). |
| `env_type` | string | No | From config | Environment type: `prod` or `non-prod`. |
| `pipeline_type` | string | No | From config | Pipeline type: `dev`, `ci`, or `cd`. Controls merge approval gate and SonarQube. |
| `target_bucket` | string | No | From config | Production S3 bucket name for final deployment. |
| `test_bucket` | string | No | From config | Test/staging S3 bucket for pre-production validation. |
| `src_bucket` | string | No | From config | Source S3 bucket (production bucket used for no-change builds). |
| `from_env` | string | No | From config | Source environment for CD pipelines (e.g., "qa" for stg). |
| `run_integration_tests` | bool | No | From config | Whether to run integration tests (controls HOOKS env var). |
| `source_repository` | string | No | `tolunaengineering/{app_name}` | Bitbucket repository path. |
| `environment_variables` | map(string) | No | `{}` | Additional environment variables passed to build CodeBuild. |
| `environment_variables_parameter_store` | map(string) | No | ADO_USER, ADO_PASSWORD | SSM parameter references for CodeBuild env vars. |
| `cypress_record_tests` | bool | No | `false` | Enable Cypress Dashboard recording. |
| `cypress_record_key` | string | No | `""` | Cypress Dashboard project record key. |
| `vpc_config` | map | No | `{vpc_id="NULL"}` | VPC configuration for build CodeBuild. |
| `security_group_ids` | list(string) | No | `[]` | Security group IDs merged into vpc_config from config. |

---

## Sub-modules

| Sub-module | Purpose | Key Resources |
| --- | --- | --- |
| `modules/build` | CodeBuild project for compiling the frontend application. Supports VPC networking, custom env vars, and templated buildspec. | `aws_codebuild_project`, `aws_iam_role`, `aws_iam_role_policy` |
| `modules/test` | CodeBuild project for running Cypress E2E tests against the test deployment. | `aws_codebuild_project`, `aws_iam_role`, `aws_iam_role_policy` |
| `modules/post` | CodeBuild project for production promotion (S3 sync + CloudFront invalidation). | `aws_codebuild_project`, `aws_iam_role`, `aws_iam_role_policy` |
| `modules/ci-cd-codepipeline` | AWS CodePipeline orchestrating all stages. | `aws_codepipeline`, `aws_iam_role`, `aws_iam_role_policy` |

---

## S3 Bucket Configuration

The module manages ownership controls on three S3 buckets, typically created in the same Terraform apply (same app layer):

| Bucket | Purpose | Ownership Control |
| --- | --- | --- |
| `target_bucket` | Production — serves live traffic via CloudFront | `BucketOwnerPreferred` (always) |
| `test_bucket` | Staging — serves test deployment for Cypress validation (public-read, website hosting enabled) | `BucketOwnerPreferred` (always) |
| `src_bucket` | Source/production bucket (reused for no-change builds to sync from) | `BucketOwnerPreferred` (non-prod only) |

Additionally, a CodePipeline artifacts bucket must pre-exist: `s3-codepipeline-{app_name}-{env_type}`

---

## CodeBuild Environment

All three CodeBuild projects share these settings:

| Setting | Value |
| --- | --- |
| Compute | `BUILD_GENERAL1_SMALL` |
| Image | `aws/codebuild/amazonlinux2-x86_64-standard:5.0` |
| Type | `LINUX_CONTAINER` |
| Privileged Mode | `true` |
| Build Timeout | 120 minutes |
| Node.js Runtime | 20 (set in buildspec install phase) |
| Logs | CloudWatch Logs (`/{codebuild_name}/log-group`) |

---

## Usage Example (from garnish-release)

```hcl
# terraform/app/pipeline.tf
module "pipeline" {
  source  = "env1.toluna.me/Garnish/terraform-aws-frontend-pipeline/aws"
  version = "3.1.0"

  config               = module.config
  distribution_id      = aws_cloudfront_distribution.spa_distribution.id
  test_distribution_id = aws_cloudfront_distribution.spa_test_distribution.id
  security_group_ids   = [aws_security_group.garnish-release_default.id]
  source_repository    = "tolunaengineering/garnish"

  depends_on = [
    aws_s3_bucket.spa_bucket,
    aws_s3_bucket.spa_test_bucket,
    module.config.frontend_pipeline
  ]
}
```

---

## Required SSM Parameters

| Parameter | Used By | Purpose |
| --- | --- | --- |
| `/app/ado_user` | Module + buildspec | Azure DevOps username for npm registry |
| `/app/ado_password` | Module + buildspec | Azure DevOps PAT for npm registry |
| `/app/codepipeline_ado_user` | Buildspec | ADO user (buildspec parameter-store) |
| `/app/codepipeline_ado_pat` | Buildspec | ADO PAT (buildspec parameter-store) |
| `/app/bb_user` | Buildspec (build) | Bitbucket username for basic auth API calls |
| `/app/bb_app_pass` | Buildspec (build) | Bitbucket app password for basic auth |
| `/app/bb_token` | Test/post buildspecs | Bitbucket OAuth token for bearer auth |
| `/infra/sonarqube/host` | Buildspec (build) | SonarQube server hostname |
| `/infra/sonarqube/token` | Buildspec (build) | SonarQube authentication token |
| `/infra/sonarqube/username` | Buildspec (build) | SonarQube username |
| `/infra/sonarqube/password` | Buildspec (build) | SonarQube password |
| `/app/jira_release_hook` | Post buildspec | Jira release automation webhook URL |
| `/infra/codepipeline/connection_arn` | Pipeline module | CodeStar connection ARN |

---

## Limitations and Known Constraints

- **Buildspec template required at Terraform root** — The build stage expects a `buildspec.yml.tpl` file at the Terraform root (not inside the module). Test and post buildspecs are included in the module's `templates/` directory.
- **CodePipeline artifacts bucket must pre-exist** — The `s3-codepipeline-{app_name}-{env_type}` bucket is referenced as a data source and must be created separately.
- **No module outputs** — The module does not export any outputs. All state is internal to the pipeline.
- **SSM parameters must be pre-configured** — All parameters listed above must exist before applying.
- **src_bucket ownership control only in non-prod** — The `BucketOwnerPreferred` setting on `src_bucket` is only applied when `env_type == "non-prod"`.
- **Cypress configs are per-environment** — Test stage expects `cypress.config.{env_name}.js` in `service/test/`. Missing config file will fail the test stage.
- **SonarQube only on non-dev** — Quality gate analysis is skipped for `pipeline_type == "dev"`.
- **Report group must be created separately** — The `{app_name}-{env_name}-IntegrationTestReport` CodeBuild report group is not created by the module.
