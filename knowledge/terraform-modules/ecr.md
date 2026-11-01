---
title: ecr
description: Creates an ECR repository for Docker images, provisioned only in shared-non-prod with cross-account pull access for prod and lifecycle-based image pruning.
tags: [plm, terraform, module, ecr, shared]
---

# ecr

Creates an ECR (Elastic Container Registry) repository for Docker images. Only provisioned in the `shared-non-prod` workspace — the prod account pulls images cross-account.

Part of [Terraform Modules](./index.md). Its `ecr_repo_url` output is read by [config-manager](./config-manager.md) and used by [lambda-gitflow](./lambda-gitflow.md) and [ecs-gitflow](./ecs-gitflow.md).

---

## Quick Reference

| Property | Value |
| --- | --- |
| **Registry** | `env1.toluna.me/toluna/terraform-aws-ecr/aws` |
| **Current Version** | 3.0.1 |
| **Repository** | [tolunaengineering/terraform-aws-ecr](https://bitbucket.org/tolunaengineering/terraform-aws-ecr) |
| **Present In** | Shared layer (non-prod only) |

---

## Purpose

Every containerized service (Lambda or ECS) needs a Docker image registry. The ECR module creates the repository, configures cross-account access for prod, and manages image lifecycle policies to prune old images.

---

## Runtime Behavior

### At terraform apply time

- Creates ECR repository named after the service (`app_name`)
- Grants cross-account pull access to the prod AWS account (so prod can pull images built in non-prod)
- Configures image lifecycle policy (via `ecr_cleaner` sub-module) to remove untagged/old images
- Only created when workspace is `shared-non-prod` (guarded by `count`)

### During deployments

- **Build stage** (lambda-gitflow/ecs-gitflow): Docker image is pushed here with tags `latest`, `{version}`, `{env_name}`
- **Pre-deploy stage**: Image tag is copied between environments (e.g., `qa` tag → `prod` tag for CD pipelines)
- **Post-deploy stage**: Image re-tagged for promotion

### Image lifecycle

The `ecr_cleaner` sub-module uses a Lambda on a schedule to enforce retention policies — removing images that are no longer referenced by any deployed function/task.

---

## Module Interactions

### Inputs From

| Source | What It Provides |
| --- | --- |
| [config-manager](./config-manager.md) | `config.ecr.repo_name` (= app_name), `config.ecr.principal` (prod account ARN for cross-account access) |

### Outputs Consumed By

| Consumer | What It Uses |
| --- | --- |
| [config-manager](./config-manager.md) (app sub-module) | Reads `ecr_repo_url` from shared remote state → provides it to lambda/ecs modules |
| [lambda-gitflow](./lambda-gitflow.md) / [ecs-gitflow](./ecs-gitflow.md) build stage | Pushes Docker images to this repo |
| [lambda-gitflow](./lambda-gitflow.md) / [ecs-gitflow](./ecs-gitflow.md) pre stage | Copies image tags between environments |

---

## Inputs

| Variable | Type | Required | Description |
| --- | --- | --- | --- |
| `config` | object | Yes | Config-manager output (provides repo name and cross-account principal) |

---

## Sub-modules

| Sub-module | Purpose |
| --- | --- |
| `ecr_cleaner` | Lambda + EventBridge schedule for image lifecycle management (prune old/untagged images) |

---

## Usage Example

```hcl
module "ecr" {
  count   = module.config.globals.environment == "shared-non-prod" ? 1 : 0
  source  = "env1.toluna.me/toluna/terraform-aws-ecr/aws"
  version = "3.0.1"
  config  = module.config
}
```

The `count` guard ensures the repo is only created once. All environments (dev, qa, stg, prod) share the same ECR repository — differentiated by image tags.
