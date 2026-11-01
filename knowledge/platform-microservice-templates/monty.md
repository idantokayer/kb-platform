---
title: Monty — Python Lambda Microservice Template
description: Toluna's Python Lambda microservice template — a health-check Lambda on Python 3.12 with Datadog layer tracing, private API Gateway (IAM auth), and the standard three-layer setup.
tags: [plm, template, microservice, lambda, python, monty]
---

# Monty — Python Lambda Microservice Template

**Tribe:** Buffet | **App Name:** `monty-release` | **App Role:** `EXP` | **App Type:** `sam` | **Runtime:** Python 3.12

Part of [Platform microservice templates](./index.md). Compare with the .NET Lambda template, [Candynet](./candynet.md).

---

## 1. Service Overview

Monty is the **Python Lambda microservice template** for Toluna. It demonstrates the standard three-layer Terraform architecture with a single health-check Lambda function that validates connectivity to DynamoDB, S3, SQS, and external HTTP endpoints.

The Lambda runs on Python 3.12, uses the Datadog Lambda layer for tracing, and is exposed via a **private** API Gateway with IAM auth (AWS SigV4).

---

## 2. Layer Breakdown

See [Terraform Infrastructure layers](../terraform-infrastructure-layers.md) for the layer model.

### Shared Layer

| Module | Version | Purpose |
| --- | --- | --- |
| [config](../terraform-modules/config-manager.md) | 3.1.1 | Central config |
| [ecr](../terraform-modules/ecr.md) | 3.0.1 | Docker image registry (non-prod only) |
| `source_pipeline_controller` ([pipeline-controller](../terraform-modules/pipeline-controller.md)) | 3.1.0 | Bitbucket webhook → CodePipeline trigger |
| [test_framework](../terraform-modules/test-framework.md) | 3.2.0 | Postman integration test runner |
| [shared-security-group](../terraform-modules/shared-security-group.md) | 3.0.1 | VPC security group for Lambda |
| [switch-traffic](../terraform-modules/switch-traffic.md) | 3.0.4 | DNS (private domain, APIGW via VPC endpoint) |
| [datadog](../terraform-modules/datadog-integration.md) | 3.0.1 | Datadog AWS integration |
| [finops](../terraform-modules/finops.md) | 3.0.0 | Cost tracking |

### App Layer

| Module | Version | Purpose |
| --- | --- | --- |
| [config](../terraform-modules/config-manager.md) | 3.1.1 | Central config |
| `lambda_health` ([lambda](../terraform-modules/lambda.md)) | 3.0.6 | Lambda function + integrations |
| `pipeline` ([lambda-gitflow](../terraform-modules/lambda-gitflow.md)) | 3.2.1 | CodePipeline (build → deploy → traffic switch) |
| [switch-traffic](../terraform-modules/switch-traffic.md) | 3.0.4 | DNS for dev workspaces only |

### Data Layer

| Resource | Details |
| --- | --- |
| DynamoDB Table | `{app_name}-{env}-MyTable`, PAY_PER_REQUEST, GSI on CurrencyId+Credits |
| S3 Bucket | `{app_name}-{environment}` |
| SQS Queue | `sqs-app_queue-{app_name}-{environment}` (visibility 900s, long-poll 5s) |
| SSM Parameter | SQS queue URL stored in Parameter Store |

---

## 3. Lambda Function — Health Check

| Property | Value |
| --- | --- |
| **Function name** | `monty-release-health-{environment}` |
| **Runtime** | Python 3.12 (container image) |
| **Handler** | `datadog_lambda.handler.handler` (wrapping real handler via `DD_LAMBDA_HANDLER`) |
| **Timeout** | 30s |
| **Memory** | 1024 MB |
| **Integrations** | API GW (`GET /health_api`), EventBridge (1-min cron), DynamoDB, SQS, S3 |

---

## 4. Python-Specific Build

### Dockerfile — Multi-Stage

```
Stage 1: build (platform-python3-base:3.12)
  ├── pip install aiohttp asyncio boto3
  ├── pytest → JUnit XML + coverage XML
  └── sonar-scanner (quality gate enforced)

Stage 2: Runtime (public.ecr.aws/lambda/python:3.12)
  ├── COPY app code + Datadog layer from build stage
  └── pip install -r requirements.txt
```

Tests and SonarQube run INSIDE the Docker build. If they fail, the image is never pushed.

---

## 5. Differences from Candynet (.NET)

| Aspect | Monty (Python) | Candynet (.NET) |
| --- | --- | --- |
| Runtime | Python 3.12 container | .NET 10 container |
| Handler pattern | Datadog wrapper via `DD_LAMBDA_HANDLER` env var | Native .NET handler |
| Test execution | `pytest` inside Docker build | `dotnet test` inside Docker build |
| Static analysis | `sonar-scanner` CLI | dotnet-sonarscanner |
| Dependencies | `pip install -r requirements.txt` | `dotnet restore` / NuGet |
| Async | `aiohttp` + `asyncio` | `async/await` native |

### Shared Infrastructure (Identical)

Both templates use the same module ecosystem: [config-manager](../terraform-modules/config-manager.md), [lambda](../terraform-modules/lambda.md), [lambda-gitflow](../terraform-modules/lambda-gitflow.md), [switch-traffic](../terraform-modules/switch-traffic.md), [pipeline-controller](../terraform-modules/pipeline-controller.md), [test-framework](../terraform-modules/test-framework.md), [ecr](../terraform-modules/ecr.md), [dynamodb](../terraform-modules/dynamodb.md). The Terraform layer structure and wiring pattern are identical. See [Candynet](./candynet.md).

---

## 6. Key Patterns for New Services

### Adding a New Lambda Function

1. Create `terraform/app/lambda_<name>.tf` with the `terraform-toluna-lambda` module
2. Wire API Gateway integration via `apigw_integration_list`
3. Add to `function_list` in `pipeline.tf`
4. Create Python handler at `service/monty/<name>/lambda_<name>_handler.py`
5. Set `DD_LAMBDA_HANDLER` to the real handler path

### EventBridge Schedule

The health Lambda runs on a 1-minute cron via `event_bridge_config`. Use this pattern for any scheduled Lambda.
