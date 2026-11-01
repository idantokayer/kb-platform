---
title: vpces
description: Provisions individual AWS VPC Endpoints (Interface and Gateway types) with associated security groups for private connectivity to AWS services.
tags: [plm, terraform, module, vpces, vpc-endpoints, networking]
---

# vpces

Provisions individual AWS VPC Endpoints (Interface and Gateway types) with associated security groups for private connectivity to AWS services.

Part of [Terraform Modules](./index.md). Release notes: [2.0.5](./vpces-2.0.5.md). Consumed by the [vpc](./vpc.md) module.

---

## Quick Reference

| Property | Value |
| --- | --- |
| **Registry** | `env1.toluna.me/toluna/terraform-aws-vpces/aws` |
| **Current Version** | 2.0.5 |
| **Repository** | [tolunaengineering/terraform-aws-vpces](https://bitbucket.org/tolunaengineering/terraform-aws-vpces) |
| **Present In** | App layer (consumed by the [vpc](./vpc.md) module) |

---

## Purpose

The vpces module creates individual VPC Endpoints for AWS services (ECS, ECR, S3, DynamoDB, SSM, SQS, Logs, Monitoring, Kinesis Firehose). It is consumed by the [vpc](./vpc.md) module which orchestrates which endpoints to create based on service requirements.

---

## Supported Endpoints

| Endpoint | Type | Variable |
| --- | --- | --- |
| ECS Agent | Interface | `create_ecs_vpce` |
| ECS Telemetry | Interface | `create_ecs_vpce` |
| ECS | Interface | `create_ecs_vpce` |
| ECR API | Interface | `create_ecr_vpce` |
| ECR DKR | Interface | `create_ecr_vpce` |
| Logs | Interface | `create_logs_vpce` |
| SSM | Interface | `create_ssm_vpce` |
| SQS | Interface | `create_sqs_vpce` |
| Monitoring | Interface | `create_monitoring_vpce` |
| S3 | Gateway | `create_s3_vpce` |
| DynamoDB | Gateway | `create_dynamodb_vpce` |
| Kinesis Firehose | Interface | `create_kinesis_firehose_vpce` |
