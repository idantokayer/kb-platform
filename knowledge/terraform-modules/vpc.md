---
title: vpc
description: Provisions the complete VPC infrastructure for a service account — VPC, subnets, NAT gateway, transit gateway attachment, and all required VPC Endpoints via the vpces sub-module.
tags: [plm, terraform, module, vpc, networking, app-layer]
---

# vpc

Provisions the complete VPC infrastructure for a service account — VPC, subnets, NAT gateway, transit gateway attachment, and all required VPC Endpoints via the vpces sub-module.

Part of [Terraform Modules](./index.md). Release notes: [2.1.2](./vpc-2.1.2.md). Orchestrates the [vpces](./vpces.md) module.

---

## Quick Reference

| Property | Value |
| --- | --- |
| **Registry** | `env1.toluna.me/toluna/terraform-aws-vpc/aws` |
| **Current Version** | 2.1.2 |
| **Repository** | [tolunaengineering/terraform-aws-vpc](https://bitbucket.org/tolunaengineering/terraform-aws-vpc) |
| **Present In** | App layer |

---

## Purpose

The vpc module is the **network foundation** for every service account. It creates the VPC, subnets, routing, NAT gateway, transit gateway attachment, and orchestrates VPC Endpoints (via the [vpces](./vpces.md) module) based on the service's requirements.

---

## Module Interactions

### Inputs From

| Source | What It Provides |
| --- | --- |
| [config-manager](./config-manager.md) | VPC CIDR, subnet configuration, endpoint toggles |

### Outputs Consumed By

| Consumer | What It Uses |
| --- | --- |
| [ecs](./ecs.md) / [lambda](./lambda.md) | VPC ID, subnet IDs, security group IDs |
| [shared-security-group](./shared-security-group.md) | VPC ID for creating service security groups |

---

## Key Variables

| Variable | Type | Description |
| --- | --- | --- |
| `create_ecs_vpce` | bool | Create ECS VPC Endpoints |
| `create_ecr_vpce` | bool | Create ECR VPC Endpoints |
| `create_s3_vpce` | bool | Create S3 Gateway Endpoint |
| `create_dynamodb_vpce` | bool | Create DynamoDB Gateway Endpoint |
| `create_ssm_vpce` | bool | Create SSM Interface Endpoint |
| `create_sqs_vpce` | bool | Create SQS Interface Endpoint |
| `create_logs_vpce` | bool | Create CloudWatch Logs Endpoint |
| `create_monitoring_vpce` | bool | Create CloudWatch Monitoring Endpoint |
| `create_kinesis_firehose_vpce` | bool | Create Kinesis Firehose Interface Endpoint |
