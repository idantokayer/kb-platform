---
title: Pipelines
description: Index of the platform CI/CD pipeline documentation — pipeline types (dev/CI/CD) and the end-to-end deployment flow.
tags: [plm, ci-cd, pipelines, index]
---

# Pipelines

Documentation of the platform's CI/CD pipelines. Part of [VII - Platform Infrastructure](../platform-infrastructure.md).

## Pages

- [Pipelines Types](./pipelines-types.md) — the three pipeline types (Dev, CI, CD), their triggers, and behavior.
- [Deployment Flow](./deployment-flow.md) — the complete deployment lifecycle for Lambda, ECS, and Frontend services, from PR to production traffic.

## Related

- [pipeline-controller](../terraform-modules/pipeline-controller.md) — the module that orchestrates the pipelines.
- [Terraform Infrastructure layers](../terraform-infrastructure-layers.md) — the blue/green layer model pipelines operate on.
