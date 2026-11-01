---
title: datadog-integration 3.1.0 — Release Notes
description: Release notes for datadog-integration 3.1.0 — migrates to the new datadog_integration_aws_account provider resource with dynamic IAM policies (breaking change).
tags: [plm, terraform, module, datadog, release-notes]
---

# 3.1.0

Release notes for the [datadog-integration](./datadog-integration.md) module.

## Release Details

| Property | Value |
| --- | --- |
| Version | 3.1.0 |
| Released | 2026-08-06 |
| Operator | Waseem Khatib |
| PR | [PR #9](https://bitbucket.org/tolunaengineering/terraform-aws-datadog-integration/pull-requests/9) |
| Ticket | [PLM-3480](https://toluna.atlassian.net/browse/PLM-3480) |
| Approver | Eli Meitner |

## What Changed

Migrated the Datadog AWS integration to the **new provider API resource** — the old `datadog_integration_aws` resource is deprecated by Datadog and this release moves to the supported `datadog_integration_aws_account` resource. IAM policies are now dynamically sourced from Datadog's API instead of a static JSON file, so they stay current without manual updates.

### Breaking Changes

- **Integration resource replaced** — `datadog_integration_aws` → `datadog_integration_aws_account`. Terraform will destroy the old resource and create a new one. The integration will be briefly disconnected during apply.
- **IAM resources replaced** — old `aws_iam_policy.datadog_integration_policy` and `aws_iam_role.datadog_integration_role` are destroyed and recreated with new names. Expect destroys + creates in plan.
- **Removed variables:** `datadog_forwarder_aws_region`, `datadog_policy_name`, `metrics_collection_enabled` — any usage of these will cause a plan error.
- **Default for `metrics_to_collect` changed** from `["lambda", "dynamodb"]` → `[]` — if you relied on the default, you must now explicitly list the namespaces you want. See the module README for the full list of available namespace values.
- **Namespace format changed** — use full AWS namespace names (e.g., `AWS/Lambda`) instead of short names (e.g., `lambda`).
- **Removed resources:** `datadog_integration_aws_lambda_arn` and `datadog_integration_aws_log_collection` — these are now configured inline within the single `datadog_integration_aws_account` resource.

### Improvements

- **IAM policies auto-update** — policies are dynamically sourced from `datadog_integration_aws_iam_permissions` data source with automatic chunking to respect AWS 6144-char policy size limits. No more manually maintaining a JSON policy file.
- **Better security posture** — added `SecurityAudit` AWS managed policy and `DatadogAccountInformationPolicy` for `account:GetAccountInformation`.
- **Log subscription filters use actual ARN** — references the Forwarder ARN from CloudFormation output instead of hardcoding function name + region. Works correctly regardless of region or naming.
- **Examples cleaned up** — all examples updated to use `config = module.config` pattern, removed hardcoded API keys, source updated to `env1.toluna.me` registry.
- **README auto-generated with tf-docs** — includes full list of available metric namespaces.

## How to Upgrade

**This is a breaking change.** Terraform will destroy and recreate the Datadog integration resources. The integration will be briefly disconnected during apply (~1-2 minutes).

This module lives in the **shared layer** (`terraform/shared/datadog.tf`). The shared layer is always applied manually — no pipeline applies it automatically.

### Steps

1. **Update `terraform/shared/datadog.tf`:**
   - Change module version to `3.1.0`
   - Remove any references to: `datadog_forwarder_aws_region`, `datadog_policy_name`, `metrics_collection_enabled`
   - If you had no explicit `metrics_to_collect` (relying on the old default), add it explicitly with full namespace names.
   - If you used short namespace names, convert to full names (e.g., `lambda` → `AWS/Lambda`)
2. **Run:**
   ```shell
   cd terraform/shared
   terraform init -upgrade
   terraform plan -target="module.datadog"
   terraform apply -target="module.datadog"
   ```
3. **Apply order:** `shared-non-prod` first → verify Datadog integration reconnects and metrics/logs flow → then `shared-prod`
4. **Expected plan output** — destroys + creates for:
   - `datadog_integration_aws.integration` → replaced by `datadog_integration_aws_account.integration`
   - `aws_iam_policy.datadog_integration_policy` → replaced by `aws_iam_policy.datadog_aws_integration[*]`
   - `aws_iam_role.datadog_integration_role` → replaced by `aws_iam_role.datadog_aws_integration`
   - `datadog_integration_aws_lambda_arn` and `datadog_integration_aws_log_collection` → removed (handled inline)
5. **Verify after apply:** Check in Datadog console that the AWS integration shows the account connected, metrics are flowing, and log forwarding is active.

## Affected Services

All services that have the `datadog` module enabled in their shared layer. Typically one "primary" service per AWS account (e.g., `bread` in buffet). Most services have this module commented out and are **not affected**.
