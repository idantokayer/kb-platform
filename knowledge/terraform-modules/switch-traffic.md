---
title: switch-traffic (DNS & Event Routing)
description: Manages DNS records, Lambda event source mappings, EventBridge schedules, SNS subscriptions, and Step Function aliases — the traffic-flip module that makes blue/green work.
tags: [plm, terraform, module, switch-traffic, dns, blue-green, shared, app-layer]
---

# switch-traffic

Manages DNS records, Lambda event source mappings, EventBridge schedules, SNS subscriptions, and Step Function aliases. The "traffic flip" module that makes blue/green deployments work.

Part of [Terraform Modules](./index.md). Invoked by [pipeline-controller](./pipeline-controller.md) during the merge build. See the [Deployment Flow](../pipelines/deployment-flow.md).

---

## Quick Reference

| Property | Value |
| --- | --- |
| **Registry** | `env1.toluna.me/toluna/terraform-toluna-switch-traffic/aws` |
| **Current Version** | 3.0.8 |
| **Repository** | [tolunaengineering/terraform-toluna-switch-traffic](https://bitbucket.org/tolunaengineering/terraform-toluna-switch-traffic) |
| **Present In** | Shared layer (always) + App layer (dev pipeline type only) |

---

## Purpose

Switch-traffic is the module that **routes live traffic** to the correct deployment. It manages DNS records, SQS→Lambda event source mappings, EventBridge schedules, SNS subscriptions, and Step Function aliases.

During a merge build, the pipeline-controller runs `terraform apply -target=module.switch-traffic` to flip all traffic sources from the old deployment to the new one.

---

## Runtime Behavior

### At terraform apply time (shared layer)

- Creates Route53 private hosted zone for the service
- Creates ACM certificates (private domain)
- Creates DNS records: A/CNAME pointing to API Gateway (via VPC endpoint ALB) or ALB or CloudFront
- Creates Lambda event source mappings (SQS → `live` alias) for all blue/green environments
- Creates EventBridge schedules pointing to `live` aliases
- Creates SNS subscriptions pointing to `live` aliases
- Creates Step Function state machines and aliases
- Uses cross-account role (`aws.net` provider) for VPC endpoint domain association via RAM sharing

### During a merge build (runtime)

The merge buildspec in pipeline-controller runs:

```
terraform apply -target=module.switch-traffic
```

This is the **traffic flip** — at this moment:

1. DNS records update to point to the new API Gateway stage / ALB
2. SQS event source mappings re-point to the new Lambda `live` alias
3. EventBridge rules re-target the new `live` alias
4. SNS subscriptions re-target the new `live` alias

All existing in-flight requests to the old DNS drain over 300 seconds (TTL).

### App layer (dev only)

For developer overlay environments (`pipeline_type == "dev"`), switch-traffic runs in the **app layer** instead of shared. It:

- Creates DNS records pointing directly to the dev API Gateway / ALB
- Creates SQS event source mappings for the dev Lambda

This is because dev environments don't use the shared blue/green mechanism.

---

## Module Interactions

### Inputs From

| Source | What It Provides |
| --- | --- |
| [config-manager](./config-manager.md) (shared) | `config.shared.lambdas_list` — all Lambda functions needing event source mappings |
| [config-manager](./config-manager.md) (shared) | `config.shared.app_records` / `config.shared.ecs_records` / `config.shared.spa_records` — DNS record definitions |
| [config-manager](./config-manager.md) (shared) | `config.shared.step_function_definition_list` — Step Function definitions |
| [config-manager](./config-manager.md) (app) | `config.net.vpce_alb`, `config.net.domain_association_role_arn` — VPC endpoint for DNS |
| App layer resources | `api_gw_id` or `alb_dns_name` (only in app layer for dev) |

### Outputs Consumed By

| Consumer | What It Uses |
| --- | --- |
| [pipeline-controller](./pipeline-controller.md) merge build | `terraform apply -target=module.switch-traffic` flips traffic |
| All service consumers | DNS records route traffic to the correct backend |
| SQS queues | Event source mappings route messages to correct Lambda version |

---

## Inputs

| Variable | Type | Required | Description |
| --- | --- | --- | --- |
| `config` | object | Yes | Config-manager output |
| `dns_input` | object | No | DNS configuration |
| `lambda_event_source_mapping_input` | list(object) | No | SQS → Lambda mappings (app layer / dev only) |
| `step_function_definition_list` | map | No | Step Function definitions |

### `dns_input` key fields

| Field | Description |
| --- | --- |
| `records_type` | `"apigw"` (Lambda), `"alb"` (ECS), or `"spa"` (CloudFront) |
| `is_private_domain` | Use private hosted zone |
| `use_vpc_endpoint_for_apigw` | Route API Gateway through VPC endpoint |
| `api_gw_id` | API Gateway ID (app layer only, required for non-shared) |
| `alb_dns_name` | ALB DNS name (app layer only, ECS services) |
| `spa_public_domain` | CloudFront domain (app layer only, SPA) |

**Validation rules:**

- In shared layer: `api_gw_id` and `alb_dns_name` must be null (reads from config instead)
- In app layer: at least one of `api_gw_id` or `alb_dns_name` must be set

---

## Sub-modules

| Sub-module | Purpose |
| --- | --- |
| `shared_dns` | Route53 zone, ACM cert, base DNS records |
| `alb_dns` | ALB-specific DNS alias records |
| `api_gw_alb_dns` | API Gateway through VPC endpoint ALB DNS |
| `api_gw_public_dns` | Public API Gateway DNS (for tribe gateways) |
| `lambda_event_source_mapping` | SQS → Lambda `live` alias mappings |
| `lambda_eventbridge_integration` | EventBridge → Lambda schedules |
| `lambda_sns_subscription` | SNS → Lambda subscriptions |
| `shared_step_functions` / `step_functions` | Step Function state machine + alias management |

Each DNS sub-module contains a `callee_domains_mapping` sub-module for cross-VPC domain resolution via RAM sharing.

---

## Usage Example

### Shared Layer (all services)

```hcl
module "switch-traffic" {
  source  = "env1.toluna.me/toluna/terraform-toluna-switch-traffic/aws"
  version = "3.0.8"

  config = module.config
  dns_input = {
    records_type               = "apigw"
    is_private_domain          = true
    use_vpc_endpoint_for_apigw = true
  }

  providers = {
    aws.net = aws.net
    aws     = aws
  }
}
```

### App Layer (dev pipelines only)

```hcl
module "switch-traffic" {
  source  = "env1.toluna.me/toluna/terraform-toluna-switch-traffic/aws"
  version = "3.0.8"
  count   = module.config.app.pipeline_type == "dev" ? 1 : 0

  config = module.config
  dns_input = {
    records_type               = "apigw"
    is_private_domain          = true
    api_gw_id                  = aws_api_gateway_rest_api.api.id
    use_vpc_endpoint_for_apigw = true
  }
  lambda_event_source_mapping_input = [local.sqs_mapping]

  providers = {
    aws.net = aws.net
    aws     = aws
  }
}
```
