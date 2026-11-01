---
title: vpces 2.0.5 — Release Notes
description: Release notes for vpces 2.0.5 — adds a Kinesis Firehose Interface VPC Endpoint and fixes the region data source attribute.
tags: [plm, terraform, module, vpces, release-notes]
---

# 2.0.5

Release notes for the [vpces](./vpces.md) module.

| Field | Value |
| --- | --- |
| **Released** | 2026-07-29 |
| **Released By** | Eli Meitner |
| **Ticket** | [PLM-3954](https://toluna.atlassian.net/browse/PLM-3954) |
| **Branch** | `master` |
| **Tag** | `2.0.5` |

---

## What Changed

- **Added Kinesis Firehose VPC Endpoint** — new `aws_vpc_endpoint.kinesis_firehose` resource with Interface type, controlled by `create_kinesis_firehose_vpce` variable
- **Fixed region data source usage** — changed all endpoint `service_name` references from `data.aws_region.current.name` to `data.aws_region.current.region` (correct attribute name)
- **Updated local endpoint name resolution** — the `endpoint_name` local now includes `kinesis_firehose`

---

## Migration Steps

If your service was using `create_sns_vpce = true`:

1. If you need Kinesis Firehose connectivity, add `create_kinesis_firehose_vpce = true`

---

## Affected Services

All services using the vpces module (typically via the [vpc](./vpc.md) module).
