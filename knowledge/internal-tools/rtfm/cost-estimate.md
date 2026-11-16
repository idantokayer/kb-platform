---
title: RTFM - Cost Estimate
description: Back-of-envelope monthly cost model for RTFM per space under bursty and steady traffic, with the caveats that matter when budgeting.
tags: [plm, internal-tools, rtfm, cost, finops]
---

# RTFM - Cost Estimate

Part of [RTFM](./index.md).

These figures are **estimates at public on-demand us-east-1 rates**, not a quote. For a budget number, price the microVM line in the AWS Pricing Calculator and read a real month in Cost Explorer grouped by SERVICE.

## Caveats

1. **microVM compute is the dominant line and the least certain.** `lambda-microvms` is a newer service whose per-vCPU-hour and per-GB-hour rate is not priced here. The figures use a Fargate-equivalent proxy (about $0.04/vCPU-hr + $0.004/GB-hr, ARM).
2. **Per-VM sizing is not settable through `RunMicrovm`.** There is no CPU or memory parameter at run time; size comes from the image (`minimumMemoryInMiB=4096`). The 2 GB to 8 GB range in the model is a cost-modeling assumption.

## Assumptions

| Parameter | Value |
|---|---|
| Spaces | 1 (`platform`) |
| Request rate | 500 req/hour (about 360k/month) |
| Modeled VM | about 2.5 GB / 1.25 vCPU average |
| Pricing | public on-demand, no committed-use discounts |

## Scenario A — bursty traffic (VM about 80% idle)

| Component | $/month |
|---|---|
| microVM compute (about 146 active hr) | ~9–12 |
| microVM suspended overhead | ~1–3 |
| API Gateway | <1 |
| Cognito (<10k MAU) | 0 |
| Lambda (proxy and authorizer) | ~1–3 |
| Lambda (LifecycleManager and TokenRotation) | <1 |
| DynamoDB (pay per request) | <1 |
| EFS | <1 |
| EventBridge / Route53 / ACM / SSM / S3 | ~1–2 |
| **Total (1 space)** | **~20–30** |

## Scenario B — steady traffic (VM always on)

500 requests/hour spread evenly keeps the VM below the idle threshold, so it effectively never suspends (730 hr/month).

| Component | $/month |
|---|---|
| microVM compute (730 hr) | ~45–55 |
| All other lines combined | ~5–10 |
| **Total (1 space)** | **~55–70** |

## Scaling by number of spaces

Only the microVM line and API Gateway scale with spaces and traffic.

| Spaces | Scenario A | Scenario B |
|---|---|---|
| 1 | ~$20–30 | ~$55–70 |
| 5 | ~$90–140 | ~$250–330 |
| 10 | ~$170–270 | ~$480–650 |

The service-level planning figure of about $10/month prod and about $2/month non-prod (single space, aggressive idle suspension) is a lower bound for mostly-idle spaces.

## Getting a real number

1. Before launch: the AWS Pricing Calculator, with the explicit `lambda-microvms` per-vCPU-hour and per-GB-hour rate.
2. After a day at about 500 req/hr: Cost Explorer month-to-date grouped by SERVICE, which includes real microVM pricing and account discounts.
