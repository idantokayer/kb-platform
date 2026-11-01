---
title: MCP Platform Operations
description: The gateway operator's runbook for the MCP platform — the current Lambda-layer version table per MCP package (deploy, hooks, and auth-wiring runbook, Platform-team owned).
tags: [plm, mcp, steering, operations, runbook]
---

# MCP Platform Operations

The gateway operator's runbook for the MCP platform (deploy, hooks, auth wiring) — owned by the Platform team. Service developers do not need it; see the [MCP Development Guide](../docs/mcp-development-guide.md) and [MCP Development Protocol](./mcp-development-protocol.md) instead.

Part of [MCP Development](../index.md) (Steering).

## Lambda Layer Versions

Current published Lambda-layer version per MCP package:

| Package | Layer Name | Current Version |
| --- | --- | --- |
| datadog-mcp-server-py | datadog-mcp-server-layer | :14 |
| usain-bolt-mcp-server-py | usain-bolt-mcp-server-layer | :5 |
| env1-mcp-server-py | env1-mcp-server-layer | :14 |
| chamberlain-mcp-server-py | chamberlain-mcp-server-layer | :8 |

Related: [Remote MCP Gateway](../docs/remote-mcp-gateway.md) (architecture), [MCP Gateway — Generic Handler & V2](../docs/mcp-gateway-v2-design.md), [Design Alternatives & Incidents](../docs/mcp-design-alternatives.md) (INC-003 stale-layer deploy, INC-005 auth wiring).
