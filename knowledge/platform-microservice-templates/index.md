---
title: Platform microservice templates
description: Reference implementations for building microservices on the Toluna Cloud Platform — Candynet, Monty, Snake, and Garnish templates with CI/CD and blue/green pre-configured.
tags: [plm, templates, microservices, index]
---

# Platform Microservice Templates

Reference implementations for building microservices on the Toluna Cloud Platform. Each template provides a fully working scaffold with CI/CD, observability, and blue/green deployment pre-configured.

Part of [VII - Platform Infrastructure](../platform-infrastructure.md).

Templates are not cloned directly — developers receive a repository based on their chosen template after going through the [new service creation process](https://toluna.atlassian.net/wiki/spaces/SOA/pages/1111588870) (external Confluence).

---

## Available Templates

| Template | Service Type | Runtime | Use When |
| --- | --- | --- | --- |
| [Candynet](./candynet.md) | Lambda (API + SQS consumer) | .NET 10 | Building a .NET backend with API Gateway + DynamoDB + SQS |
| [Monty](./monty.md) | Lambda (API) | Python 3.12 | Building a Python backend with API Gateway + DynamoDB |
| [Snake](./snake.md) | ECS Fargate | Python 3.12 (FastAPI) | Building a long-running Python service with ALB routing |
| [Garnish](./garnish.md) | Frontend SPA | Node.js (TypeScript) | Building a single-page application served via CloudFront + S3 |

---

## What Each Template Includes

- **Terraform infrastructure** — All three layers (shared, app, data) pre-wired with the standard module ecosystem. See [Terraform Infrastructure layers](../terraform-infrastructure-layers.md).
- **CI/CD pipeline** — Blue/green deployment via CodeDeploy (Lambda/ECS) or S3 sync (SPA). See the [Deployment Flow](../pipelines/deployment-flow.md).
- **Observability** — Datadog APM tracing, structured logging, and metrics pre-configured
- **Quality gates** — SonarQube analysis and unit tests enforced during Docker build
- **Multi-environment support** — Dev overlays, QA, staging, and production workspaces ready to go

---

## How to Get a Template

1. Submit a new service request through the [new service creation process](https://toluna.atlassian.net/wiki/spaces/SOA/pages/1111588870) (external Confluence)
2. Choose the template that matches your service type
3. The platform team provisions the repository, AWS accounts, and infrastructure
4. You receive a ready-to-use repository with all layers pre-configured

See each child page for the full architecture breakdown, module versions, and wiring details.
