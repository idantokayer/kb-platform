---
title: shared-security-group
description: Creates a common VPC security group shared by all Lambda functions and ECS services in an account, kept in the immutable shared layer to speed up blue/green deployments.
tags: [plm, terraform, module, security-group, networking, shared]
---

# shared-security-group

Creates a common VPC security group shared by all Lambda functions and ECS services in an account. Avoids creating per-function security groups.

Part of [Terraform Modules](./index.md). Its SG ID is exposed via [config-manager](./config-manager.md) as `config.app.shared_security_group`.

---

## Quick Reference

| Property | Value |
| --- | --- |
| **Registry** | `env1.toluna.me/toluna/terraform-aws-shared-security-group/aws` |
| **Current Version** | 3.0.1 |
| **Repository** | [tolunaengineering/terraform-aws-shared-security-group](https://bitbucket.org/tolunaengineering/terraform-aws-shared-security-group) |
| **Present In** | Shared layer |

---

## Purpose

Instead of each Lambda/ECS service creating its own security group in the app layer, a single shared SG is created in the **shared (immutable) layer**. This serves two goals:

1. **Simplifies VPC rules** — avoids hitting security group limits and reduces complexity
2. **Speeds up deployments** — because the SG lives in the shared layer (which is rarely destroyed), it avoids the slow AWS networking APIs during blue/green workspace creation and destruction. Creating/deleting ENIs, detaching security groups, and cleaning up subnet associations are notoriously slow operations. By keeping the SG in an immutable layer, the app layer's `terraform apply` and `terraform destroy` skip these expensive networking calls entirely.

The `is_ecs` flag controls the CIDR rules (ECS needs broader inbound access than Lambda).

---

## Runtime Behavior

### At terraform apply time

- Creates one `aws_security_group` in the service's VPC
- For Lambda (`is_ecs = false`): allows outbound to all, inbound restricted
- For ECS (`is_ecs = true`): adds inbound from `10.0.0.0/16` (VPC-internal traffic from ALB)
- The SG ID is stored in shared layer remote state

### At deploy time

- Lambda functions and ECS tasks are placed in this security group (resolved by config-manager from shared remote state)
- No runtime changes — the SG rules are static
- Because the SG already exists and is never destroyed/recreated during deployments, there's no waiting for ENI detachment or SG deletion — the app layer just references it

---

## Module Interactions

### Inputs From

| Source | What It Provides |
| --- | --- |
| [config-manager](./config-manager.md) | `config.net.vpc_id` for SG creation |

### Outputs Consumed By

| Consumer | What It Uses |
| --- | --- |
| [config-manager](./config-manager.md) (app sub-module) | Reads SG ID from shared remote state → exposes as `config.app.shared_security_group` |
| [lambda](./lambda.md) module | Uses SG for Lambda VPC configuration |
| [ecs](./ecs.md) module | Uses SG for ECS task networking |
| CodeBuild projects (test-framework, pipeline) | Uses SG for VPC-attached builds |

---

## Inputs

| Variable | Type | Required | Description |
| --- | --- | --- | --- |
| `config` | object | Yes | Config-manager output (provides VPC ID) |

The `is_ecs` flag is resolved internally from `config.app_type` — no manual input needed.

---

## Usage Example

```hcl
module "shared-security-group" {
  source  = "env1.toluna.me/toluna/terraform-aws-shared-security-group/aws"
  version = "3.0.1"
  config  = module.config
}
```
