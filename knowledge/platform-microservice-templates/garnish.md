---
title: Garnish — Frontend SPA Template
description: Toluna's microservice template for Single Page Applications — S3 + CloudFront (live + test), no data layer, deployed via the frontend-pipeline module with Cypress E2E.
tags: [plm, template, microservice, spa, frontend, cloudfront, garnish]
---

# Garnish — Frontend SPA Template

**Tribe:** Buffet | **App Name:** `garnish-release` | **App Role:** `FE` | **App Type:** `spa`

Part of [Platform microservice templates](./index.md). Deployed by the [frontend-pipeline](../terraform-modules/frontend-pipeline.md) module.

---

## 1. Service Overview

Garnish is the Toluna microservice template for **Single Page Applications (SPAs)**. Unlike backend templates, garnish has **no data layer** — only shared and app layers. The deployment target is S3 + CloudFront instead of ECS/Fargate, and there is no ECR, no CodeDeploy, and no ALB.

---

## 2. Layer Breakdown

See [Terraform Infrastructure layers](../terraform-infrastructure-layers.md).

### Shared Layer

| Module | Version | Purpose |
| --- | --- | --- |
| [config](../terraform-modules/config-manager.md) | 3.0.1 | Central config (`app_type = "spa"`) |
| `switch_traffic` ([switch-traffic](../terraform-modules/switch-traffic.md)) | 3.0.3 | DNS records — `records_type = "spa"`, `spa_public_domain` |
| `source_pipeline_controller` ([pipeline-controller](../terraform-modules/pipeline-controller.md)) | 3.0.3 | Bitbucket webhook → pipeline trigger |

Also creates: CodePipeline S3 bucket, IAM role, per-environment S3 folders.

### App Layer

| Module | Version | Purpose |
| --- | --- | --- |
| [config](../terraform-modules/config-manager.md) | 3.0.1 | Same config module, app-layer context |
| `pipeline` ([frontend-pipeline](../terraform-modules/frontend-pipeline.md)) | 3.0.0 | CodeBuild + CodePipeline for SPA build/deploy |

Plus directly defined resources:

- 2x CloudFront distributions (live + test)
- 2x S3 buckets (live + test)
- IAM role, security group, CodeBuild report group

---

## 3. CloudFront Distribution Setup

### Live Distribution

| Setting | Value |
| --- | --- |
| Origin | S3 bucket via OAI |
| Default root object | `index.html` |
| Error handling | 403 → 200 `/index.html` (SPA client-side routing) |
| Cache | GET/HEAD only, compress, default TTL 3600s |
| SSL | ACM wildcard cert, TLSv1.2, SNI-only |
| Price class | `PriceClass_100` (US, Canada, Europe) |

### Test Distribution

Identical but points to the **test S3 bucket**. Used for pre-approval validation.

### S3 Buckets

| Bucket | Access | Purpose |
| --- | --- | --- |
| Live | Private, OAI-only, versioned | Production traffic via CloudFront |
| Test | Public-read, website-enabled | Pre-approval testing |

---

## 4. Pipeline Flow

```
Bitbucket PR merge → Pipeline Controller → CodePipeline:
  1. BUILD: npm install → npm build → npm test → SonarQube → tag
  2. DEPLOY TO TEST S3 → Invalidate test CloudFront
  3. TEST: Cypress E2E against test CloudFront URL
  4. MANUAL APPROVAL: Reviewer validates on test URL
  5. DEPLOY TO LIVE S3 → Invalidate live CloudFront
  6. POST: Finalize, notify, Bitbucket status
```

See the full [Deployment Flow](../pipelines/deployment-flow.md).

---

## 5. Key Differences from Backend Templates

| Aspect | Backend (ECS/Lambda) | Frontend (Garnish) |
| --- | --- | --- |
| Layers | shared → data → app | shared → app (no data layer) |
| Compute | ECS/Fargate or Lambda | None — static files |
| Container registry | ECR | None |
| Serving | ALB → ECS or API GW → Lambda | CloudFront → S3 |
| Deployment | CodeDeploy blue/green | S3 sync + CloudFront invalidation |
| DNS target | ALB or API Gateway | CloudFront distribution |
| Build output | Docker image | Static files (HTML/JS/CSS) |
| Test environment | Blue/green task set or Lambda alias | Test S3 + test CloudFront |
| switch_traffic | `records_type` = "alb" or "apigw" | `records_type = "spa"` |

---

## 6. Environment Config Strategy

Each environment has a dedicated config file:

```
service/configs/
├── envConfigs.js           # Default/local
├── envConfigs.devops.js    # DevOps environment
├── envConfigs.qa.js        # QA
└── envConfigs.prod.js      # Production
```

At build time: `cp ./configs/envConfigs.${ENV_NAME}.js ./build/configs/envConfigs.js`

The SPA is built once but configured per environment via this config file swap.

---

## 7. Blue/Green for Frontend

1. **Test phase ("green"):** Build artifacts → test S3 → test CloudFront
2. **Manual approval:** Reviewer validates on test URL
3. **Live deploy ("promote"):** Artifacts → live S3 → CloudFront invalidation
4. **Rollback:** S3 versioning — restore previous object versions

No true zero-downtime — CloudFront cache propagation creates a brief window (typically <60s).

---

## 8. Domain Structure

```
Non-prod:
  Live:  {env}.garnish-release-non-prod.tolunastart.com → CloudFront (live)
  Test:  {env}-test.garnish-release-non-prod.tolunastart.com → CloudFront (test)

Prod:
  Live:  garnish-release.tolunastart.com → CloudFront (live)
  Test:  test.garnish-release.tolunastart.com → CloudFront (test)
```
