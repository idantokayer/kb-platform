---
title: test-framework
description: Creates integration test infrastructure — CodeBuild report groups and a test-framework-manager Lambda that orchestrates Postman/Newman runs during CodeDeploy deployments.
tags: [plm, terraform, module, test-framework, testing, ci-cd, shared]
---

# test-framework

Creates the integration test infrastructure: CodeBuild report groups and a test-framework-manager Lambda that orchestrates Postman/Newman test runs during CodeDeploy deployments.

Part of [Terraform Modules](./index.md). Report groups are consumed by [lambda-gitflow](./lambda-gitflow.md) and [ecs-gitflow](./ecs-gitflow.md).

---

## Quick Reference

| Property | Value |
| --- | --- |
| **Registry** | `env1.toluna.me/toluna/terraform-aws-codepipline-test-framework/aws` |
| **Current Version** | 3.4.0 |
| **Repository** | [tolunaengineering/terraform-aws-codepipline-test-framework](https://bitbucket.org/tolunaengineering/terraform-aws-codepipline-test-framework) |
| **Present In** | Shared layer |

---

## Purpose

Provides the integration testing layer for the CI/CD pipeline. Creates CodeBuild report groups (unit tests, code coverage, integration tests) consumed by the gitflow pipeline stages, and a Lambda that runs Postman collections against deployed services during the CodeDeploy validation phase.

---

## Runtime Behavior

### At terraform apply time

- Creates 3 CodeBuild report groups: `TestReport`, `CodeCoverageReport`, `IntegrationTestReport`
- Creates `test-framework-manager` Lambda with VPC access
- Creates `integration-runner` CodeBuild project for Postman/Newman execution
- Creates S3 bucket for test collections and results
- Creates IAM roles for test execution

### During deployments (runtime)

1. CodeDeploy fires the `AfterAllowTestTraffic` hook (ECS) or can be triggered by merge_waiter
2. `test-framework-manager` Lambda is invoked
3. Lambda starts the `integration-runner` CodeBuild project
4. CodeBuild pulls Postman collections (from Postman API or S3)
5. Runs Newman against the test endpoint (e.g., port 4443 for ECS green fleet)
6. Results published to `IntegrationTestReport` report group
7. Pass/fail determines if deployment proceeds or rolls back

### Report groups

The report group ARNs are passed via config-manager remote state to the gitflow modules, which reference them in their CodeBuild projects:

- **TestReport** — unit test results (JUnit XML format)
- **CodeCoverageReport** — code coverage (Cobertura XML format)
- **IntegrationTestReport** — Postman/Newman results (JUnit XML format)

---

## Module Interactions

### Inputs From

| Source | What It Provides |
| --- | --- |
| [config-manager](./config-manager.md) | `config.globals` (app_name, environments), `config.net` (VPC for test runner Lambda) |

### Outputs Consumed By

| Consumer | What It Uses |
| --- | --- |
| [config-manager](./config-manager.md) (app sub-module) | Reads report group ARNs from shared remote state |
| [lambda-gitflow](./lambda-gitflow.md) / [ecs-gitflow](./ecs-gitflow.md) | CodeBuild projects reference `TestReport`, `CodeCoverageReport` for build reports |
| [pipeline-controller](./pipeline-controller.md) merge_waiter | Can invoke test-framework-manager instead of just waiting |

---

## Inputs

| Variable | Type | Required | Description |
| --- | --- | --- | --- |
| `config` | object | Yes | Config-manager output |
| `postman_collections` | list(object) | No | Postman collection/environment IDs to run |
| `environment_variables` | map(string) | No | Extra env vars for test CodeBuild |
| `environment_variables_parameter_store` | map(string) | No | SSM parameter references for secrets |
| `tribe_vpcs` | map(object) | No | VPC config for cross-tribe test access |

---

## Sub-modules

| Sub-module | Purpose |
| --- | --- |
| `test_framework_manager` | Lambda: orchestrates test execution, invokes integration runner |
| `integration_runner` | CodeBuild project: runs Postman/Newman collections, publishes results |

---

## Usage Example

```hcl
module "test_framework" {
  source  = "env1.toluna.me/toluna/terraform-aws-codepipline-test-framework/aws"
  version = "3.4.0"
  config  = module.config
}
```
