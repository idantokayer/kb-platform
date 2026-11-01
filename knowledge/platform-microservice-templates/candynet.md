---
title: Candynet — Lambda .NET Microservice Template
description: Toluna's reference template for .NET Lambda microservices — API Gateway, SQS consumer, DynamoDB, blue/green via CodeDeploy, Datadog APM, and SonarQube gates.
tags: [plm, template, microservice, lambda, dotnet, candynet]
---

# Candynet — Lambda .NET Microservice Template Analysis

**Tribe:** Buffet | **App Name:** `candynet-release` | **App Role:** `EXP` | **App Type:** `sam` | **Runtime:** .NET 10

Part of [Platform microservice templates](./index.md). Compare with the Python Lambda template, [Monty](./monty.md).

---

## 1. Service Overview

Candynet is Toluna's **reference template for .NET Lambda microservices**. It demonstrates the standard patterns for:

- API Gateway → Lambda integration (REST, private endpoint)
- SQS → Lambda event-source mapping (queue consumer)
- DynamoDB read/write from Lambda
- Blue/green deployment via CodeDeploy
- Datadog APM integration (tracing, git metadata)
- SonarQube quality gate enforcement
- AppConfig feature flags
- Multi-tenant DynamoDB table naming
- Environment hierarchy configuration system

It is NOT a production service — it's a scaffold that teams clone to create new microservices within the Toluna platform module ecosystem.

---

## 2. Layer Breakdown

Candynet follows Toluna's standard **three-layer Terraform architecture** ([Terraform Infrastructure layers](../terraform-infrastructure-layers.md)). Each layer is an independent Terraform root module with its own state file, workspace, and apply lifecycle.

### 2.1 Data Layer

**Purpose:** Provisions persistent, stateful resources that survive app redeployments.

| Resource | Name Pattern | Details |
| --- | --- | --- |
| SQS Queue | `sqs-app_queue-candynet-release-{env}` | Visibility timeout 900s, long-poll 1s |
| SSM Parameter | `/infra/candynet-release-{env}/sqs_queue_url` | Stores SQS URL for cross-layer discovery |
| DynamoDB Table(s) | `candynet-release-{env}-MyTable` (single-tenant) or `candynet-release-{env}-{tenant}-MyTable` (multi-tenant) | PAY_PER_REQUEST billing, primary key `DataId` (N), GSI on `CurrencyId`+`Credits` |

### 2.2 Shared Layer

**Purpose:** Provisions account-level and cross-environment resources.

| Module | Version | Purpose |
| --- | --- | --- |
| [config](../terraform-modules/config-manager.md) | 3.1.3 | Central config |
| [switch-traffic](../terraform-modules/switch-traffic.md) | 3.0.5 | Private API GW domain via VPC endpoint |
| [ecr](../terraform-modules/ecr.md) | 3.0.1 | Container image registry (non-prod only) |
| `source_pipeline_controller` ([pipeline-controller](../terraform-modules/pipeline-controller.md)) | 3.1.3-PLM-3698 | Bitbucket webhook → pipeline trigger |
| [test_framework](../terraform-modules/test-framework.md) | 3.2.1 | Integration test runner |
| [shared-security-group](../terraform-modules/shared-security-group.md) | 3.0.1 | Common SG for Lambda VPC access |
| [finops](../terraform-modules/finops.md) | 3.0.0 | Cost tagging |

### 2.3 App Layer

**Purpose:** Provisions Lambda functions, API Gateway, pipeline, and per-environment resources.

| Module | Version | Purpose |
| --- | --- | --- |
| [config](../terraform-modules/config-manager.md) | 3.1.1 | Central config |
| `pipeline` ([lambda-gitflow](../terraform-modules/lambda-gitflow.md)) | 3.2.1 | CodePipeline CI/CD |
| [switch-traffic](../terraform-modules/switch-traffic.md) | 3.0.5-PLM-3701 | DNS for dev pipelines only |
| `lambda_sqs_listener` ([lambda](../terraform-modules/lambda.md)) | 3.0.6 | SQS consumer Lambda |
| `lambda_sqs_api` ([lambda](../terraform-modules/lambda.md)) | 3.0.6 | API → SQS writer Lambda |
| `lambda_create_dynamo_api` ([lambda](../terraform-modules/lambda.md)) | 3.0.6 | API → DynamoDB writer Lambda |
| `lambda_create_health_api` ([lambda](../terraform-modules/lambda.md)) | 3.0.6 | Health check Lambda |

---

## 3. Lambda Functions

| # | Function Name Pattern | Trigger | Integrations |
| --- | --- | --- | --- |
| 1 | `candynet-release-sqs_app_trigger-{env}` | SQS event source mapping (batch=10, concurrency=3) | SQS read, DynamoDB write |
| 2 | `candynet-release-sqs_api-{env}` | API GW `POST /WriteDataToSQS` | SQS write |
| 3 | `candynet-release-dynamo_api-{env}` | API GW `POST /WriteDataToDynamo` | DynamoDB write |
| 4 | `candynet-release-health_api-{env}` | API GW `GET /health_api` | SQS + DynamoDB (connectivity check) |

---

## 4. API Gateway

- **Type:** Private REST API (VPC endpoint only)
- **Endpoint:** `PRIVATE` (no public access)
- **Policy:** Allows `execute-api:Invoke` from any principal with VPC access
- **Routes:** `/candynet-release` → POST /WriteDataToSQS, POST /WriteDataToDynamo, GET /health_api

---

## 5. Key Patterns for New Services

### Adding a new Lambda function

1. Create handler class in `service/APIGatewayLambda/` or `service/SQSListenerLambda/`
2. Add `lambda_{name}.tf` in `terraform/app/` using the `terraform-toluna-lambda` module
3. Add function name to `local.lambda_list` in `main.tf`
4. If API-triggered: add `apigw_integration_list` block pointing to the API GW resource
5. If SQS-triggered: add event source mapping config and wire to `switch-traffic`

### Adding a new DynamoDB table

1. Add table base name to `local.table_types` in `terraform/data/dynamodb.tf`
2. Add new `module "dynamodb_{name}"` block with `for_each` on the decoded table names
3. Multi-tenant naming is automatic when `is_multitenant = true`

### Developer overlay environment

1. Add `{name}.json` to `service/Shared/ConfigurationFiles/`
2. Add `"{name}": "COMMON"` to `EnvironmentHierarchy.json`
3. Create Terraform workspace: `terraform workspace new {name}` in app and data layers
4. Pipeline type auto-detected as `dev`

---

## 6. Notable Design Decisions

1. **Private API Gateway only** — No public endpoints. Security relies on VPC-level network isolation.
2. **Single Docker image, multiple handlers** — All Lambda functions share one container image. The handler string selects which class to invoke.
3. **SQS partial batch failure** — `ReportBatchItemFailures` response type means individual message failures don't retry the entire batch.
4. **Blue/green with shared data** — Blue and green Lambda versions share the same SQS queue and DynamoDB tables.
5. **Feature flags via AppConfig** — Environments created per workspace in shared layer.
