---
title: datadog-integration
description: Sets up the full Datadog↔AWS integration — cross-account IAM role, Datadog Forwarder Lambda, CloudWatch log subscriptions, metrics collection, and an optional Lambda dashboard.
tags: [plm, terraform, module, datadog, observability, shared]
---

# datadog-integration

Sets up a full Datadog ↔ AWS integration: cross-account IAM role, Datadog Forwarder Lambda (via CloudFormation), CloudWatch log subscriptions, metrics collection configuration, and an optional Lambda performance dashboard.

Part of [Terraform Modules](./index.md). Release notes: [3.1.0](./datadog-integration-3.1.0.md). Configured by [config-manager](./config-manager.md).

---

## Quick Reference

| Property | Value |
| --- | --- |
| **Registry** | `env1.toluna.me/toluna/terraform-aws-datadog-integration/aws` |
| **Current Version** | 3.1.0 |
| **Repository** | [tolunaengineering/terraform-aws-datadog-integration](https://bitbucket.org/tolunaengineering/terraform-aws-datadog-integration) |
| **Present In** | Shared layer (one per AWS account, typically enabled in a single "primary" service) |
| **Providers** | AWS + `DataDog/datadog` v3.81.0 |

---

## Purpose

Establishes the Datadog ↔ AWS integration for an account. This is a **one-time per account** setup that:

1. Allows Datadog to read AWS metrics via a cross-account IAM role
2. Deploys a Lambda Forwarder that ships CloudWatch logs to Datadog
3. Subscribes specific log groups to the forwarder
4. Configures which AWS service namespaces Datadog collects metrics from
5. Optionally creates a Datadog dashboard for Lambda performance

In most services, this module is **commented out** — only the account's "primary" service (e.g., `bread` in the buffet account) enables it.

---

## Runtime Behavior

### At terraform apply time

1. **Stores Datadog API key in Secrets Manager** — `datadog_api_key` secret with zero-day recovery (immediate delete on destroy)
2. **Deploys Forwarder Lambda via CloudFormation** — uses Datadog's official template (`latest.yaml`), passes API key ARN, site, tags, and log exclusion patterns
3. **Creates integration IAM role** — `DatadogAWSIntegrationRole` trusted by Datadog's account (`464622532012`) with external ID condition
4. **Attaches integration policies** — dynamic IAM policy documents sourced from `datadog_integration_aws_iam_permissions` data source, automatically chunked to respect AWS policy size limits, plus `SecurityAudit` managed policy and `AccountInformation` policy
5. **Registers AWS integration with Datadog API** — uses `datadog_integration_aws_account` resource to configure account ID, role, metrics, logs, and resource collection
6. **Subscribes CloudWatch log groups** — attaches subscription filters pointing to the forwarder Lambda ARN (from CloudFormation output)
7. **Optionally creates Lambda dashboard** — only for non-prod when `metrics_to_collect` is non-empty

### At runtime

- **Metrics:** Datadog assumes the integration IAM role every ~10 minutes to scrape CloudWatch metrics from configured namespaces
- **Logs:** CloudWatch log subscription filters invoke the Forwarder Lambda on every log event → Lambda ships logs to Datadog
- **APM Traces:** Not handled by this module — traces flow directly from the Datadog Lambda layer/sidecar to Datadog's intake endpoint

### Plan-time API call

The module calls `https://api.datadoghq.com/api/v1/integration/aws/available_namespace_rules` at **plan time** (via `data.http`) to fetch the list of available metric namespaces. If Datadog API is unreachable or keys are invalid, `terraform plan` will fail.

---

## Module Interactions

### Inputs From

| Source | What It Provides |
| --- | --- |
| [config-manager](./config-manager.md) | `config.datadog.dd_api_key`, `config.datadog.dd_app_key`, `config.datadog.host_tags` |
| [config-manager](./config-manager.md) | `config.globals.app_name`, `config.globals.env_type` (for dashboard conditional) |
| Service log groups (external) | Passed via `cloudwatch_log_groups` or `cloudwatch_log_groups_as_list` |

### Outputs Consumed By

| Consumer | What It Uses |
| --- | --- |
| [ecs-gitflow](./ecs-gitflow.md) post-deploy | Subscribes new ECS log groups to the Forwarder Lambda (`datadog-forwarder`) by name convention |
| Lambda/ECS services | Datadog Lambda layer / sidecar sends traces to Datadog; Forwarder ships logs |
| Datadog platform | Assumes integration role to read metrics; receives logs from Forwarder |

---

## Inputs

| Variable | Type | Required | Description |
| --- | --- | --- | --- |
| `config` | object | Yes | Config-manager output (provides `datadog.*`, `globals.app_name`, `globals.env_type`) |
| `dd_api_key` | string | No | Override Datadog API key (default: from config) |
| `dd_app_key` | string | No | Override Datadog APP key (default: from config) |
| `dd_site` | string | No | Datadog site URL (default: `datadoghq.com`) |
| `dd_tags` | string | No | Custom tags for forwarded logs, comma-delimited |
| `create_datadog_forwarder` | bool | No | Deploy the Forwarder Lambda (default: `true`) |
| `metrics_to_collect` | list | No | AWS namespaces to enable (e.g., `["AWS/Lambda", "AWS/ECS"]`). Default: `[]` |
| `cloudwatch_log_groups` | map | No | Log groups to subscribe (map with `.name` attribute) |
| `cloudwatch_log_groups_as_list` | map | No | Log groups to subscribe (map keyed by log group name) |
| `exclude_logs_pattern` | string | No | Regex to exclude from forwarded logs (default: excludes START/REPORT/END/TELEMETRY/EXTENSION) |
| `host_tags` | list(string) | No | Tags added to all hosts/metrics (format: `key:value`) |
| `filter_tags` | list(string) | No | EC2 tag filters for metric collection |
| `aws_regions` | list | No | AWS regions to include (default: `["us-east-1"]`) |
| `log_collection_services` | list | No | AWS services for automatic log collection |
| `extended_resource_collection_enabled` | string | No | Enable extended resource collection (default: from Datadog) |

---

## Usage Example

```hcl
# Only enable in one service per account (e.g., "bread" in buffet)
module "datadog" {
  source  = "env1.toluna.me/toluna/terraform-aws-datadog-integration/aws"
  version = "3.1.0"
  config  = module.config

  metrics_to_collect = ["AWS/Lambda", "AWS/ECS", "AWS/SQS", "AWS/DynamoDB"]

  cloudwatch_log_groups_as_list = {
    "/ecs/bread-release-qa" = {}
    "/ecs/bread-release-stg" = {}
  }
}
```
