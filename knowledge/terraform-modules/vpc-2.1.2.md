---
title: vpc 2.1.2 — Release Notes
description: Release notes for vpc 2.1.2 — adds Kinesis Firehose VPC Endpoint support and bumps the internal vpces module to 2.0.5.
tags: [plm, terraform, module, vpc, release-notes]
---

# 2.1.2

Release notes for the [vpc](./vpc.md) module.

| Field | Value |
| --- | --- |
| **Released** | 2026-07-29 |
| **Released By** | Eli Meitner |
| **Ticket** | [PLM-3954](https://toluna.atlassian.net/browse/PLM-3954) |
| **Branch** | `master` |
| **Tag** | `2.1.2` |

---

## What Changed

- **Added Kinesis Firehose VPC Endpoint support** — new `create_kinesis_firehose_vpce` boolean variable (default: `false`) that provisions a Kinesis Firehose Interface endpoint via the [vpces](./vpces.md) sub-module
- **Bumped vpces module version** — updated internal vpces module reference from `2.0.3` to `2.0.5`
- **Fixed region data source usage** — changed all vpces module `region` arguments from `data.aws_region.current.name` to `data.aws_region.current.region` (correct attribute name)

---

## Migration Steps

No migration required — drop-in upgrade. The new `create_kinesis_firehose_vpce` variable defaults to `false`, so existing services are unaffected.

To enable the Kinesis Firehose endpoint for a service, add:

```hcl
create_kinesis_firehose_vpce = true
```

---

## Affected Services

All services using the vpc module can upgrade at their next deployment. No action required unless Kinesis Firehose VPC Endpoint connectivity is needed.
