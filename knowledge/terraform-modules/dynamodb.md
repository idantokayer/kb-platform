---
title: dynamodb
description: Creates DynamoDB tables with multi-tenant support, autoscaling, Global Secondary Indexes, and optional data lake export (Firehose to Iceberg).
tags: [plm, terraform, module, dynamodb, data-layer]
---

# dynamodb

Creates DynamoDB tables with multi-tenant support, autoscaling, Global Secondary Indexes, and optional data lake export (Firehose → Iceberg).

Part of [Terraform Modules](./index.md). Lives in the data layer — see [Terraform Infrastructure layers](../terraform-infrastructure-layers.md). Configured by [config-manager](./config-manager.md).

---

## Quick Reference

| Property | Value |
| --- | --- |
| **Registry** | `env1.toluna.me/toluna/terraform-aws-dynamodb/aws` |
| **Current Version** | 3.0.4 |
| **Repository** | [tolunaengineering/terraform-aws-dynamodb](https://bitbucket.org/tolunaengineering/terraform-aws-dynamodb) |
| **Present In** | Data layer |

---

## Purpose

Creates DynamoDB tables for microservices with support for single-tenant and multi-tenant naming patterns. Handles autoscaling, GSI creation, and optional data lake export to Iceberg tables via Firehose.

---

## Runtime Behavior

### At terraform apply time

- Creates DynamoDB table(s) with specified schema (primary key, sort key, GSIs)
- Multi-tenant mode: creates one table **per tenant** using `for_each` on `toluna_envs` from config
- Single-tenant mode: creates one table with `"default"` key
- Naming pattern: `{app_name}-{env}-{table}` (single) or `{app_name}-{env}-{tenant}-{table}` (multi)
- Billing mode: PAY_PER_REQUEST (on-demand) by default
- Optional: autoscaling policies (target tracking on consumed read/write capacity)

### Configuration source

The dynamodb module pulls its settings via `config-manager`, which reads them from `data-env.json` located in the env1 workspace management system (S3 config bucket: `toluna-{tribe}-tf-config-data/{app_name}/data-env.json`). Multi-tenant mode, capacity settings, and autoscaling configuration are all managed there.

### At service runtime

- Lambda functions access tables using the constructed name (injected via env vars or SSM)
- Table names are discovered by the app layer via data layer remote state outputs (`table_names`)
- Config-manager resolves `data_env_name` by stripping blue/green suffix — so `qa-blue` and `qa-green` both access tables in `qa-data` workspace

### Data lake export (optional)

The `export_to_datalake` sub-module can set up DynamoDB Streams → Firehose → S3 (Iceberg format) for analytics export. Not enabled by default.

---

## Module Interactions

### Inputs From

| Source | What It Provides |
| --- | --- |
| [config-manager](./config-manager.md) | `config.data.is_multitenant`, `config.globals.env_vars.toluna_envs` (tenant list), `config.globals.environment` |
| `data-env.json` (via config-manager) | Multi-tenant flag, capacity settings, autoscaling config |

### Outputs Consumed By

| Consumer | What It Uses |
| --- | --- |
| [config-manager](./config-manager.md) (app sub-module) | Reads `table_names` from data remote state → passed to Lambda env vars |
| Lambda functions (runtime) | Access tables by constructed name |

---

## Inputs

| Variable | Type | Required | Description |
| --- | --- | --- | --- |
| `config` | object | Yes | Config-manager output |
| `app_name` | string | Yes | Used in table name construction |
| `table_name` | string | Yes | Full table name (constructed by caller using `for_each`) |
| `primary_key` | string | Yes | Hash key attribute name |
| `primary_key_type` | string | Yes | Hash key type: `S`, `N`, or `B` |
| `billing_mode` | string | No | `PAY_PER_REQUEST` (default) or `PROVISIONED` |
| `global_secondary_indeces` | list(object) | No | GSI definitions (hash_key, range_key, projection) |
| `sort_key` | string | No | Range key attribute name |
| `sort_key_type` | string | No | Range key type |

---

## Sub-modules

| Sub-module | Purpose |
| --- | --- |
| `export_to_datalake` | DynamoDB Streams → Firehose → S3 Iceberg for analytics export |

---

## Usage Example

```hcl
locals {
  is_multitenant = module.config.data.is_multitenant
  table_names = (
    local.is_multitenant ?
    jsonencode({ for t in ["MyTable"] : t => {
      for k in module.config.globals.env_vars.toluna_envs : k => "${local.app_name}-${module.config.globals.environment}-${k}-${t}"
    }}) :
    jsonencode({ for t in ["MyTable"] : t => {
      "default" = "${local.app_name}-${module.config.globals.environment}-${t}"
    }})
  )
}

module "dynamodb_my_table" {
  source  = "env1.toluna.me/toluna/terraform-aws-dynamodb/aws"
  version = "3.0.4"

  config     = module.config
  for_each   = jsondecode(local.table_names)["MyTable"]
  app_name   = local.app_name
  table_name = each.value

  primary_key      = "DataId"
  primary_key_type = "N"
  billing_mode     = "PAY_PER_REQUEST"

  global_secondary_indeces = [{
    name           = "GSI-1"
    hash_key       = "CurrencyId"
    hash_key_type  = "N"
    range_key      = "Credits"
    range_key_type = "N"
  }]
}
```
