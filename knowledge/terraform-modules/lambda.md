---
title: lambda
description: Creates Lambda functions with all integrations (API Gateway, ALB, SQS, EventBridge, SNS), IAM roles, VPC config, Datadog tracing, and the live alias used for blue/green.
tags: [plm, terraform, module, lambda, app-layer]
---

# lambda

Creates Lambda functions with all integrations: API Gateway, ALB, SQS, EventBridge, SNS. Manages IAM roles, VPC configuration, Datadog tracing, and the `live` alias used for blue/green deployments.

Part of [Terraform Modules](./index.md). Deployed by [lambda-gitflow](./lambda-gitflow.md); wired for traffic by [switch-traffic](./switch-traffic.md).

---

## Quick Reference

| Property | Value |
| --- | --- |
| **Registry** | `env1.toluna.me/toluna/terraform-toluna-lambda/aws` |
| **Current Version** | 3.1.2 |
| **Repository** | [tolunaengineering/terraform-toluna-lambda](https://bitbucket.org/tolunaengineering/terraform-toluna-lambda) |
| **Present In** | App layer |

---

## Purpose

The standard way to create Lambda functions on the platform.

Handles function creation, IAM roles, VPC configuration, all integration types (API GW, ALB, SQS, EventBridge, SNS), Datadog tracing env vars, warmer scheduling, and the `live` alias that CodeDeploy shifts during blue/green deployments.

---

## Runtime Behavior

### At terraform apply time

- Creates the Lambda function with a container image from ECR (image URI resolved by config-manager)
- Creates a `live` alias pointing to the latest published version
- Wires integrations: API Gateway methods, ALB target groups, SQS event source mappings, EventBridge rules, SNS subscriptions
- All integrations reference `{function_arn}:live` — not a specific version number

### During deployment (triggered by lambda-gitflow)

- The `pre` stage of lambda-gitflow updates the function code (`update-function-code`) and publishes a new version
- CodeDeploy shifts the `live` alias from the old version to the new version
- Because all integrations reference `:live`, traffic automatically flows to the new code once the alias shifts
- The warmer (if enabled) keeps instances warm via EventBridge scheduled invocations

### SQS event source mapping behavior

- When created in the lambda module directly (via `lambda_sqs_integration` sub-module), the mapping is attached to the function (not the alias)
- For blue/green SQS consumers, the event source mapping is managed by [switch-traffic](./switch-traffic.md) instead (pointing to the `live` alias)

---

## Module Interactions

### Inputs From

| Source | What It Provides |
| --- | --- |
| [config-manager](./config-manager.md) | `config.app.lambda_create_common` (image URI, VPC, security groups, Datadog env vars, tags) |
| API Gateway resources | `rest_api_id`, `parent_api_gateway_resource_id` (created inline in service's `apigw.tf`) |
| Data layer (indirect) | SQS queue URL via `data.aws_sqs_queue` lookup — injected as env var |

### Outputs Consumed By

| Consumer | What It Uses |
| --- | --- |
| [lambda-gitflow](./lambda-gitflow.md) | `aws_lambda_out` → `function_list` for CodeDeploy deployment groups |
| [switch-traffic](./switch-traffic.md) (shared) | Lambda function name, event source mapping config from config-manager's shared sub-module |
| API Gateway deployment | Integration output changes trigger redeployment hash |

---

## Inputs

| Variable | Type | Required | Description |
| --- | --- | --- | --- |
| `config` | object | Yes | Config-manager output |
| `function_name` | string | Yes | Lambda function name |
| `lambda_handler` | string | Yes | Handler path (e.g., `MyHandler::HandleAsync`) |
| `timeout` | number | Yes | Timeout in seconds |
| `memory_size` | number | Yes | Memory in MB |
| `storage_size` | number | No | Ephemeral storage (default: 512 MB) |
| `lambda_environmnet_variables` | map | No | Custom environment variables |
| `apigw_integration_list` | list(object) | No | API Gateway route integrations |
| `api_calls_to_sqs_enabled` | bool | No | Grant SQS IAM permissions |
| `api_calls_to_dynamodb_enabled` | bool | No | Grant DynamoDB IAM permissions |
| `api_calls_to_firehose_enabled` | bool | No | Grant Firehose IAM permissions |
| `api_calls_to_events_enabled` | bool | No | Grant EventBridge IAM permissions |
| `sns_integration_list` | list(string) | No | SNS topic ARNs to subscribe to |
| `hot_instance_count` | number | No | Warmer instances (default: 0) |
| `event_bridge_config` | object | No | EventBridge schedule (e.g., `{ schedule_expression = "cron(0/5 * * * ? *)" }`) |
| `reserved_concurrent_executions` | number | No | Concurrency limit |
| `logs_retention` | number | No | CloudWatch log retention days (default: 3) |
| `s3_buckets_names_array_that_can_access` | list(string) | No | S3 buckets for IAM access |

---

## Sub-modules

| Sub-module | Purpose |
| --- | --- |
| `lambda_core_creation` | Function, IAM role, `live` alias, VPC config, Datadog env vars, CloudWatch log group |
| `lambda_apigw_integration` | API Gateway resource, method, integration, permission — one per route |
| `lambda_alb_integration` | ALB target group, listener rule, target group attachment |
| `lambda_sqs_integration` | SQS event source mapping directly on the function |
| `event_bridge_integration` | EventBridge scheduler rule + Lambda target (for warmers or cron) |
| `sns_integration_list` | SNS topic subscription + invoke permission |

---

## Usage Example

```hcl
module "lambda_sqs_api" {
  source  = "env1.toluna.me/toluna/terraform-toluna-lambda/aws"
  version = "3.1.2"

  config        = module.config
  function_name = "${local.app_name}-sqs_api-${module.config.globals.environment}"
  timeout       = 30
  memory_size   = 1024
  lambda_handler = "APIGatewayLambda::APIGatewayLambda.SQSInsertDataLambdaHandler::HandleAsync"

  api_calls_to_sqs_enabled = true

  lambda_environmnet_variables = {
    SQS_URL = data.aws_sqs_queue.sqs_queue.url
  }

  apigw_integration_list = [{
    rest_api_id                    = aws_api_gateway_rest_api.api.id
    parent_api_gateway_resource_id = aws_api_gateway_resource.app_api.id
    http_method                    = "POST"
    path_part                      = "WriteDataToSQS"
    authorization                  = "NONE"
    is_api_key_required            = false
  }]
}
```
