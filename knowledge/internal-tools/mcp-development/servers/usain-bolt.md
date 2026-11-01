---
title: Usain Bolt MCP Server
description: The Usain Bolt MCP server — E2E test reports, execution, environment health, run comparison, and failed-component analysis with root-cause classification, on the usain-bolt profile.
tags: [plm, mcp, server, usain-bolt, testing]
---

# Usain Bolt MCP Server

Part of [MCP Development → Servers](../index.md). The underlying platform is [Usain Bolt](../../usain-bolt.md).

| Field | Value |
| --- | --- |
| **Repo** | [usain-bolt-mcp-server-py](https://bitbucket.org/tolunaengineering/usain-bolt-mcp-server-py) |
| **Package** | `usain-bolt-mcp-server-py` v0.1.6 (v0.2.0 in progress) |
| **Layer** | usain-bolt-mcp-server-layer:2 (VPC-enabled) |
| **Profile** | `usain-bolt` → `mcp.toluna.me/plm/usain-bolt` |
| **Auth** | Pattern B (RequestContext — caller identity) |

## Tools (10)

### Existing (v0.1.6 — live)

- `list-reports` — List E2E test reports with filtering
- `get-report` — Detailed results with automatic log analysis
- `get-test-logs` — Console, network, Datadog logs per test
- `get-environment-health` — Health summary per environment
- `execute-run` — Trigger a new test run
- `get-available-tags` — Tags available per suite/category
- `get-version` — Server version info

### New (v0.2.0 — branch: feature/compare-runs)

- `compare-runs` — Compare 2+ test runs by UUID. Returns regressed, fixed, flaky tests with their failed component. Skips in-progress runs.
- `get-failed-components` — Extract which product components are failing. Single UUID = per-test detail; multiple UUIDs = aggregated summary.
- `get-component-health` — Top failed components across N recent reports. Analyzes every failed test's step-level data + network logs + console errors. Returns a ranked table with failure count, affected tests, the failed sub-step, API errors, and root-cause classification.

## get-component-health — Usage

```
suite: "Tolunastart"
environment: "PROD"
category: "pulseTesting"   (default)
limit: 50                  (how many reports to analyze, max 50)
top: 20                    (how many entries to return)
```

**Output per entry:** `component`, `sub_step`, `failure_count`, `runs_affected`, `test_names`, `network_errors`, `console_errors`, `root_cause`, `root_cause_detail`.

## Failed Components — Extraction Rules

Aligned with the Bolt dashboard PulseSystem: only `status === "Done"` AND `failed_tests > 0` reports qualify; steps qualify if they have a `step:` marker and a failed/broken status with no warning marker; the last failed step from each bucket (testBody, afterHooks) separately — max 2 components per test; component = text before the FLOW keyword (fallback: first comma-delimited token); dedup lowercased, trimmed, tokens sorted.

## Root Cause Classification

| Classification | Meaning | Evidence |
| --- | --- | --- |
| 🔴 product_issue | Backend API returning errors | 5xx or 4xx in network logs |
| 🟡 likely_product | UI element not appearing | waitFor/timeout in step name, no selector issue |
| 🟠 likely_automation | Selector/locator problem | locator/selector in step name |
| ⚪ infra_or_product | Timeout errors | Could be slow backend or test infra |
| ❓ unknown | Insufficient data | No network/console errors found |

## Terminology Mapping

| MCP Parameter | Bolt API Field | Examples |
| --- | --- | --- |
| `suite` | `tribe` | Tolunastart, Projects, Questionnaire |
| `category` | `test_path` | tests, pulseTesting, newBuildRegression |
| `tag`/`tags` | `tests_tag` | Define, Creation, TrafficAndTA |
| `environment` | `environment` | PROD, TRN, QAC, QAB, STG (uppercase) |

## Key Bug Fixes

- **Category vs Tag** — `category` = folder path, `tag` = filter within it.
- **User Identity** — proxy now passes full email (Pattern B: `username` → `owners`).
- **MCP SDK v2 upgrade** — `server.py` migrated to `MCPServer` + `@app.tool()` decorator.
- **Component extraction** — aligned with Bolt rules: last step per bucket, `step:` marker only, warning filter.

## Operational Notes

- VPC required (usain.toluna.me is private)
- 30s timeout per API call. get-component-health with 50 reports: ~30–60s total.
- MCP SDK: v2.0.0 (MCPServer class with @app.tool() decorator)
