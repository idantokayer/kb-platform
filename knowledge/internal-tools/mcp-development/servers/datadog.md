---
title: Datadog MCP Server
description: The Datadog MCP server — search/aggregate/correlate logs and APM traces with a 6-strategy multi-tenant filter cascade, live on the remote gateway as the datadog profile.
tags: [plm, mcp, server, datadog, observability]
---

# Datadog MCP Server

Part of [MCP Development → Servers](../index.md).

| Field | Value |
| --- | --- |
| **Repo** | [datadog-mcp-server](https://bitbucket.org/tolunaengineering/datadog-mcp-server) |
| **Package** | `datadog-mcp-server-py` v0.7.5 |
| **Layer** | datadog-mcp-server-layer:13 |
| **Profile** | `datadog` → `mcp.toluna.me/plm/datadog` |

## Tools

- `search-logs` — Search logs with filtering (query, time range, indexes, tenant)
- `aggregate-logs` — Analytical queries and aggregations on log data
- `search-traces` — Search APM traces with multi-tenant filtering
- `aggregate-traces` — Aggregations on trace/span data
- `correlate-traces` — Cross-service trace correlation (intersect, exclude, direct)
- `get-version` — Server version info

## Multi-Tenant Filtering (6-Strategy Cascade)

The `tenant` parameter enables filtering by Toluna environment in shared services:

1. **span_tag** — `@TolunaEnvironment:{tenant}` (requires `DD_TRACE_HEADER_TAGS` on service)
2. **env_field** — `env:{tenant}` (legacy monoliths only)
3. **url_pattern** — `@http.url:*tolunastart-{tenant}*` (outbound calls)
4. **trace_correlation** — sample traces, check downstream spans
5. **log_correlation** — correlate via `@dd_span_id`
6. **fallback** — returns unfiltered with `confidence: "low"`

Response includes `strategies_attempted` for transparency. STG exclusion added for non-prod tenants.

**Recommended:** Add `DD_TRACE_HEADER_TAGS=TolunaEnvironment:TolunaEnvironment` to all service env vars for reliable Strategy 1.

## correlate-traces Operators

- `intersect` — traces hitting BOTH queries (find downstream errors)
- `exclude` — traces hitting A but NOT B (find skipped dependencies)
- `direct` — verify direct parent-child calls

Use limit 100+ for rare events.

## Lessons Learned

- **ARM64 compatibility** — use `arch -arm64 python3` on Apple Silicon if pydantic_core fails
- **Stdout pollution kills MCP** — all logging to stderr only
- **Split test suites** — unit (mock) and integration (real API) in separate files
- **Nested filter support** — both `{"query": "..."}` and `{"filter": {"query": "..."}}` work
