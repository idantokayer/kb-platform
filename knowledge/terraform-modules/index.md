---
title: Terraform Modules
description: Index of all Terraform modules maintained by the platform team, grouped by layer, with links to each module's reference page.
tags: [plm, terraform, modules, index]
---

# Terraform Modules

The complete reference for all Terraform modules maintained by the platform team. This is where developers find documentation on how to use our modules in their services.

Part of [VII - Platform Infrastructure](../platform-infrastructure.md).

## What each child page represents

A single Terraform module with purpose, architecture overview, usage examples (HCL snippets), inputs/outputs reference, dependencies on other modules, and limitations. Each module page may have a **Release Notes** child page tracking version history.

## Cross-cutting references

- [Module Dependency Graph](./module-dependency-graph.md) — how all modules depend on each other, from configuration through deployment.
- [Terraform Infrastructure layers](../terraform-infrastructure-layers.md) — the shared/app/data layer model these modules live in.

## Modules by layer

### Foundation (all layers)

- [config-manager](./config-manager.md) — central configuration hub; the root of the module dependency tree.

### Shared layer

- [pipeline-controller](./pipeline-controller.md) — CI/CD orchestration: webhooks, PR/merge builds, triggering, rollback, notifications.
- [switch-traffic](./switch-traffic.md) — DNS records, event source mappings, and the blue/green "traffic flip".
- [ecr](./ecr.md) — Docker image registry (created once in shared-non-prod).
- [test-framework](./test-framework.md) — integration test infrastructure and report groups.
- [shared-security-group](./shared-security-group.md) — common VPC security group.
- [finops](./finops.md) — budget alerts and cost anomaly detection.
- [datadog-integration](./datadog-integration.md) — AWS↔Datadog integration (one per account).
- [event-driven-communication](./event-driven-communication.md) — cross-tribe EventBridge bus and SNS topic.
- [vpc](./vpc.md) — VPC networking.
- [vpces](./vpces.md) — VPC endpoints.

### App layer

- [lambda](./lambda.md) — Lambda functions with all integrations and the `live` alias.
- [lambda-gitflow](./lambda-gitflow.md) — CI/CD pipeline for Lambda services.
- [ecs](./ecs.md) — ECS Fargate services.
- [ecs-gitflow](./ecs-gitflow.md) — CI/CD pipeline for ECS services.
- [frontend-pipeline](./frontend-pipeline.md) — CI/CD pipeline for frontend SPA services.
- [http-relay](./http-relay.md) — HTTP relay module.

### Data layer

- [dynamodb](./dynamodb.md) — DynamoDB tables.
