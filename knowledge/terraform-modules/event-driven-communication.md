---
title: event-driven-communication
description: Sets up cross-tribe event-driven communication — a publisher EventBridge bus, an SNS topic for incoming events, cross-account forwarding to a central hub, and CloudWatch logging.
tags: [plm, terraform, module, event-driven, eventbridge, sns, cross-tribe]
---

# event-driven-communication

Sets up cross-tribe event-driven communication infrastructure: a publisher EventBridge bus, an SNS topic for incoming events, cross-account event forwarding to a central hub, and full CloudWatch logging.

Part of [Terraform Modules](./index.md). Architecture reference: [Event Driven Architecture](https://toluna.atlassian.net/wiki/spaces/SOA/pages/2345566212) (external Confluence).

---

## Quick Reference

| Property | Value |
| --- | --- |
| **Registry** | `env1.toluna.me/toluna/terraform-aws-event-driven-communication/aws` |
| **Current Version** | 3.1.2 |
| **Repository** | [tolunaengineering/terraform-aws-event-driven-communication](https://bitbucket.org/tolunaengineering/terraform-aws-event-driven-communication) |
| **Present In** | Integration/messaging layer (per tribe account) |

---

## Purpose

Enables asynchronous, event-driven communication between tribes using a hub-and-spoke model. Each tribe publishes events to its own EventBridge bus, which forwards them to a **Central Event Bus** in the Toluna Main account (`014931512072`). The central hub then routes events to the correct subscriber SNS topics in other tribes.

This module provisions the **publisher side** — one instance per tribe per environment.

---

## Architecture

```
Tribe A Account                              Shared Account (014931512072)
┌─────────────────────────────┐              ┌────────────────────────────┐
│                             │              │                            │
│  Service publishes event    │              │   central-hub-{env}        │
│  to publisher event bus     │──forwards──→ │   (EventBridge)            │
│  {tribe}-{env}-publisher-   │  all events  │                            │
│  event-bus                  │              │   Routes by source prefix: │
│                             │              │   "qab.*" → tribe SNS      │
│                             │              │   "qac.*" → tribe SNS      │
│  SNS: {tribe}-{env}-       │◄─────────────│                            │
│  incoming-events            │  fan-out     │                            │
│  (receives from central     │  to tribe    │                            │
│   hub / other tribes)       │              └────────────────────────────┘
└─────────────────────────────┘
```

### Event Source Convention

Events are published with `source = "{bus_env_name}.{tribe_name}"` (e.g., `qab.legacy`, `trn.buffet`).

The central hub routes by `source_prefix`:

- `source_prefix = "qab"` → routes to `{tribe}-qab-incoming-events`
- `source_prefix = "trn"` → routes to `{tribe}-trn-incoming-events`

---

## Runtime Behavior

### At terraform apply time

1. **Creates publisher EventBridge bus** — `{tribe}-{env}-publisher-event-bus` with full-detail CloudWatch logging
2. **Creates CloudWatch log group** — captures all events flowing through the bus at `/aws/vendedlogs/events/event-bus/{bus-name}`
3. **Creates forwarding rule** — catches ALL events on the bus (prefix match on empty string = catch-all) and forwards to `central-hub-{bus_env_name}` in account `014931512072`
4. **Creates cross-account IAM role** — allows EventBridge to put events on the central hub bus and publish to local SNS
5. **Creates SNS topic** — `{tribe}-{env}-incoming-events` for receiving events FROM other tribes (via central hub fan-out)
6. **Sets bus access policy** — allows any account in the same OU to publish events, and the owning account to manage rules

### At service runtime

- Services publish events to the tribe's publisher event bus using `events:PutEvents`
- The forwarding rule automatically relays ALL events to the central hub (no filtering at source)
- The central hub applies routing rules and fans events to subscriber tribe SNS topics
- Subscriber services consume from their tribe's incoming-events SNS topic (typically via SQS subscription)

---

## Module Interactions

### Inputs From

| Source | What It Provides |
| --- | --- |
| Terraform workspace name | `tribe_name`, `env_name` (derived from workspace if not explicit) |
| AWS Organizations | `org_id`, `ou_id` for access policy scoping |
| Explicit variable | `ou_id` (required), `bus_env_name` (optional override) |

### Outputs Consumed By

| Consumer | What It Uses |
| --- | --- |
| Services (Lambda/ECS) | `publisher_event_bus_arn` — target for `events:PutEvents` calls |
| [switch-traffic](./switch-traffic.md) module | Can route Lambda event source mappings to SNS topic |
| Central hub (external) | Receives forwarded events from this bus |
| Subscriber services | Subscribe to `sns_topic_arn` (incoming-events) via SQS |
| [config-manager](./config-manager.md) (shared) | Stores `publisher_event_bus_arn` and `sns_topic_arn` in shared VPC config for cross-layer discovery |

---

## Inputs

| Variable | Type | Required | Description |
| --- | --- | --- | --- |
| `ou_id` | string | Yes | AWS Organizations OU ID for access scoping |
| `log_level` | string | No | EventBridge bus log level (default: `"ERROR"`) |
| `env_name` | string | No | Environment name override. Derived from workspace if null |
| `tribe_name` | string | No | Tribe name override. Derived from workspace if null |
| `bus_env_name` | string | No | Central hub bus environment suffix override. Defaults to `env_name`. Use when env_name differs from hub naming (e.g., env `qab` maps to hub `qa`) |

---

## Outputs

| Output | Description |
| --- | --- |
| `publisher_event_bus_name` | Name of the publisher EventBridge bus |
| `publisher_event_bus_arn` | ARN of the publisher EventBridge bus |
| `sns_topic_arn` | ARN of the incoming events SNS topic |
| `sns_topic_name` | Name of the incoming events SNS topic |
| `publisher_log_group_name` | CloudWatch log group name for bus events |
| `org_id` | AWS Organization ID |
| `ou_id` | AWS Organization Unit ID |

---

## Deployed Environments

| Account | Environments | Status |
| --- | --- | --- |
| 014931512072 (Toluna Main) | Central hub (QA, STG, PROD) | ✅ Deployed |
| 093601949415 (tefups-nonprod) | QAB-T, QAC-T, TRN-T | ✅ Deployed |
| 227874955675 | QAB | ✅ Deployed |
| 933562100971 | QAC | ✅ Deployed |
| 840810307322 | TRN | ✅ Deployed |
| TEFups-prod | STG, PROD | ⏳ Pending |

---

## Usage Example

```hcl
module "event_driven_communication" {
  source  = "env1.toluna.me/toluna/terraform-aws-event-driven-communication/aws"
  version = "3.1.2"

  ou_id        = "ou-xxxx-xxxxxxxx"
  env_name     = "qab"
  bus_env_name = "qa"       # maps to central-hub-qa
  tribe_name   = "buffet"
  log_level    = "ERROR"
}
```

---

## Related

- [Event Driven Architecture](https://toluna.atlassian.net/wiki/spaces/SOA/pages/2345566212) — parent architecture page (external Confluence)
- [Cross Tribe Event Driven Communication](https://toluna.atlassian.net/wiki/spaces/SOA/pages/2345598979) — detailed cross-tribe routing design (external Confluence)
