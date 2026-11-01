---
title: finops
description: Creates AWS cost monitoring resources — budget alerts, Cost Explorer anomaly detection, and cost allocation tagging with SNS/SES notifications.
tags: [plm, terraform, module, finops, cost, budgets, shared]
---

# finops

Creates AWS cost monitoring resources: budget alerts, anomaly detection, and cost allocation tagging for the service.

Part of [Terraform Modules](./index.md). Configured by [config-manager](./config-manager.md) from `shared-env.json`.

---

## Quick Reference

| Property | Value |
| --- | --- |
| **Registry** | `env1.toluna.me/toluna/terraform-aws-finops/aws` |
| **Current Version** | 3.0.0 |
| **Repository** | [tolunaengineering/terraform-aws-finops](https://bitbucket.org/tolunaengineering/terraform-aws-finops) |
| **Present In** | Shared layer |

---

## Purpose

Provides cost visibility and alerting per service. Creates AWS Budgets for spending thresholds, Cost Explorer anomaly monitors for unexpected spikes, and SES/SNS notifications when budgets are breached.

---

## Runtime Behavior

### At terraform apply time

- Creates `aws_budgets_budget` with configured threshold
- Creates `aws_ce_anomaly_monitor` for cost anomaly detection on the service's resources
- Creates `aws_ce_anomaly_subscription` to alert on detected anomalies
- Creates SNS topic + subscription for budget alert emails
- Creates SES email identity for notification sender
- Creates a Lambda for custom cost processing/reporting
- The `create_account_level_items` flag controls whether account-wide resources (like the anomaly monitor) are created — only one service per account should set this to `true`

### Configuration source

The finops module pulls its settings via `config-manager`, which reads them from `shared-env.json` located in the env1 workspace management system (S3 config bucket: `toluna-{tribe}-tf-config-data/{app_name}/shared-env.json`). Budget thresholds, alert recipients, and anomaly detection settings are all managed there.

### At runtime

- AWS Budgets evaluates spend against threshold daily
- Cost Anomaly Detection monitors for unexpected spending patterns
- When threshold breached or anomaly detected → email notification via SNS/SES
- No impact on service operation — purely observational

---

## Module Interactions

### Inputs From

| Source | What It Provides |
| --- | --- |
| [config-manager](./config-manager.md) | `config.globals` (app_name, tribe for resource tagging and budget scoping) |
| `shared-env.json` (via config-manager) | Budget thresholds, alert email recipients, anomaly detection config |

### Outputs Consumed By

No other modules consume finops outputs. It's a leaf module — purely for cost observability.

---

## Inputs

| Variable | Type | Required | Description |
| --- | --- | --- | --- |
| `config` | object | Yes | Config-manager output |
| `create_account_level_items` | bool | No | Create account-wide cost resources (default: `false`). Set `true` for exactly one service per account. |

---

## Usage Example

```hcl
module "finops" {
  source  = "env1.toluna.me/toluna/terraform-aws-finops/aws"
  version = "3.0.0"
  config  = module.config

  # Only one service per account should set this to true
  create_account_level_items = false
}
```
