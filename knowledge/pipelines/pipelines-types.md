---
title: Pipelines Types
description: The three pipeline types (Dev, CI, CD) — their purpose, triggers, Terraform/build/versioning behavior, and env1 config settings.
tags: [plm, ci-cd, pipelines, dev, blue-green]
---

# Pipelines Types

Part of [Pipelines](./index.md). See the [Deployment Flow](./deployment-flow.md) and [pipeline-controller](../terraform-modules/pipeline-controller.md).

## Pipeline Trigger Scope

The pipeline is triggered **only** when changes are introduced in:

- `service/` folder — application code changes
- `terraform/app/` folder — app-layer infrastructure changes

Changes in `terraform/data/` or `terraform/shared/` **do not trigger the pipeline**.

These layers are applied manually or via dedicated automation, since they represent persistent (data) and immutable (shared) infrastructure that doesn't participate in the blue/green deployment cycle. See [Terraform Infrastructure layers](../terraform-infrastructure-layers.md).

## Type Comparison

|  | **Dev** | **CI** | **CD** |
| --- | --- | --- | --- |
| Purpose | Feature/developer environment for testing before Staging/QA deployment. | QA/Staging deployment. | Production deployment. |
| Type | Infrastructure changes done manually by running `terraform apply`. The Dev pipeline runs after code pushed to pipeline branch. | Blue/Green (wait for merge). The infrastructure and the CI will be done automatically. The application will be **built** during the CI pipeline. | Blue/Green (wait for merge). The infrastructure done automatically. The application will be **pulled** from QA env. |
| Trigger | Push | Pull-Request | Pull-Request |
| Terraform | 🚫 | Blue/Green (infra) | Blue/Green (infra) |
| Build | ✅ | ✅ | 🚫 |
| Versioning | 🚫 | ✅ | 🚫 |
| Deployed version | **feature** branch (HEAD) | **develop** branch merged with **feature** branch; tag image with **version** and **QA** | Image taken from CI (**QA tag**); **develop** is merged into **master** |

## Config settings (env1)

**Dev:**

```json
"waseem": {
  "pipeline_type": "dev",
  "pipeline_branch": "feature/xxx",
  "from_env": "waseem",
  "is_managed_env": false,
  "run_integration_tests": false
}
```

**CI:**

```json
"qa": {
  "pipeline_type": "ci",
  "pipeline_branch": "develop",
  "from_env": "qa",
  "is_managed_env": true,
  "run_integration_tests": true
}
```

**CD:**

```json
"prod": {
  "pipeline_type": "cd",
  "pipeline_branch": "master",
  "from_env": "qa",
  "is_managed_env": true,
  "run_integration_tests": true
}
```
