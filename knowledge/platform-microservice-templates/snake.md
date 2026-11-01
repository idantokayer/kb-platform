---
title: Snake — Python ECS Fargate Template
description: Toluna's reference template for Python microservices on ECS Fargate — FastAPI + Datadog APM, internal ALB with blue/green target groups, and the ecs-gitflow pipeline.
tags: [plm, template, microservice, ecs, fargate, python, snake]
---

# Snake — Python ECS Fargate Template

**Tribe:** Buffet | **App Name:** `snake-release` | **App Role:** `EXP` | **App Type:** `ecs` | **Runtime:** Python 3.12 (Alpine)

Part of [Platform microservice templates](./index.md). Contrast with the Lambda templates ([Monty](./monty.md), [Candynet](./candynet.md)).

---

## 1. Service Overview

Snake is Toluna's reference template for Python microservices running on **ECS Fargate**. It ships a minimal FastAPI application with Datadog APM tracing (`ddtrace-run`), JSON structured logging, and a full CI/CD pipeline using the [ecs-gitflow](../terraform-modules/ecs-gitflow.md) module with blue/green CodeDeploy.

**Runtime stack:** Python 3.12 (Alpine) → FastAPI + Uvicorn → ddtrace-run wrapper → ECS Fargate (256 CPU / 512 MiB) + Datadog sidecar.

---

## 2. Layer Breakdown

See [Terraform Infrastructure layers](../terraform-infrastructure-layers.md).

### Shared Layer

| Module | Version | Purpose |
| --- | --- | --- |
| [config](../terraform-modules/config-manager.md) | 3.1.3 | Central config (`app_type = "ecs"`) |
| [ecr](../terraform-modules/ecr.md) | 3.0.2 | Docker registry (non-prod only) |
| `source_pipeline_controller` ([pipeline-controller](../terraform-modules/pipeline-controller.md)) | 3.1.0 | Bitbucket webhook → CodePipeline trigger |
| [test_framework](../terraform-modules/test-framework.md) | 3.2.0 | Postman integration test runner |
| [shared-security-group](../terraform-modules/shared-security-group.md) | 3.0.1 | Shared SG with `is_ecs = true`, CIDR `10.0.0.0/16` |
| [switch-traffic](../terraform-modules/switch-traffic.md) | 3.0.4 | DNS: `records_type = "alb"`, `is_private_domain = true` |
| [finops](../terraform-modules/finops.md) | 3.0.0 | Cost tracking |

### App Layer

| Resource / Module | Version | Purpose |
| --- | --- | --- |
| `module.config` ([config-manager](../terraform-modules/config-manager.md)) | 3.1.0 | Per-env config |
| `module.ecs` ([ecs](../terraform-modules/ecs.md)) | 3.0.1 | ECS cluster, service, task def, Datadog sidecar |
| `module.pipeline` ([ecs-gitflow](../terraform-modules/ecs-gitflow.md)) | 3.2.0 | CodePipeline + CodeDeploy blue/green |
| `module.switch-traffic` ([switch-traffic](../terraform-modules/switch-traffic.md)) | 3.0.4 | App-layer DNS (dev pipeline type only) |
| `aws_alb.private` | — | Internal ALB |
| `aws_alb_target_group.blue_fleet` | — | Blue target group (port 80, IP type) |
| `aws_alb_target_group.green_fleet` | — | Green target group (port 80, IP type) |
| `aws_alb_listener.https` (443) | — | Production listener → blue |
| `aws_alb_listener.http_test` (4443) | — | Test listener → green |

---

## 3. ECS Service Configuration

| Setting | Value |
| --- | --- |
| Launch type | Fargate |
| CPU | 256 (0.25 vCPU) |
| Memory | 512 MiB |
| App container memory | 256 MiB (soft limit) |
| Entrypoint | `ddtrace-run python3 app.py` (Uvicorn on port 80) |
| Health check | `/ping` → "Pong!" |
| Datadog sidecar | Auto-injected by ECS module |

---

## 4. ALB / Networking

- **ALB:** Internal, private subnets, shared security group
- **Target Groups:** Blue (production) + Green (test), IP target type, health check on `/ping`
- **Listeners:** 443 (HTTPS, production → blue) + 4443 (HTTPS, test → green)
- **CodeDeploy:** Swaps listener target groups during blue/green deployment

---

## 5. Pipeline Flow (ECS Gitflow)

```
Bitbucket PR merge → pipeline_controller → CodePipeline:
  1. BUILD: docker build (pytest + sonar inside) → ECR push
  2. CODEDEPLOY: Deploy green task set → test on 4443 → shift 443 to green
  3. POST: Finalize, notify, Bitbucket status
```

See the full [Deployment Flow](../pipelines/deployment-flow.md).

---

## 6. Key Differences from Lambda Template

| Aspect | Lambda (Monty/Candynet) | ECS (Snake) |
| --- | --- | --- |
| Compute | Lambda functions | ECS Fargate tasks |
| Networking | API Gateway (private) | Internal ALB + target groups |
| Deployment | Lambda alias shifting | CodeDeploy blue/green with ALB listener swap |
| Traffic split | Lambda weighted aliases | Two ALB target groups + two listeners |
| Health checks | N/A (Lambda managed) | ALB health check on `/ping` (10s interval) |
| Scaling | Lambda concurrency (auto) | ECS auto-scaling (opt-in, commented out in template) |
| Sidecar | N/A | Datadog agent sidecar container |
| Pipeline module | [lambda-gitflow](../terraform-modules/lambda-gitflow.md) | [ecs-gitflow](../terraform-modules/ecs-gitflow.md) |
| DNS | `records_type = "apigw"` | `records_type = "alb"` |
| Test listener | N/A (test via alias) | Port 4443 → green fleet |
| Cost model | Pay per invocation | Pay per running task (always-on) |

---

## 7. Observability

- **APM:** `ddtrace-run` auto-instruments FastAPI
- **Logs:** JSON structured via `python-json-logger` with `dd.trace_id` / `dd.span_id`
- **Sidecar:** `DD_APM_IGNORE_RESOURCES = "GET /ping"` filters health check noise
- **Runtime metrics:** `DD_RUNTIME_METRICS_ENABLED = true`
- **CloudWatch:** Log group `snake-release-{environment}`
