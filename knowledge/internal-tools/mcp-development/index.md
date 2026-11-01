---
title: MCP Development
description: The internal development project for Toluna's MCP platform — how to build, deploy, and operate MCP tool servers, with docs, steering, per-server pages, and skills.
tags: [plm, internal-tools, mcp, development, index]
---

# MCP Development

**Owner:** Platform Engineering Team · **Status:** Active

Part of [Internal Tools](../index.md). The user-facing setup guide is [MCP Remote Tools](../mcp-remote-tools.md).

## What Is This?

This is the **internal development project** for Toluna's MCP (Model Context Protocol) platform. It covers how we build, deploy, and operate MCP tool servers — both local and remote.

**This is NOT the user-facing guide.** For end-user setup instructions, see [MCP Remote Tools — AI Agent Configuration](../mcp-remote-tools.md).

## Project Structure

| Folder | Contents |
| --- | --- |
| **docs/** | Technical reference — how to build MCP servers, remote gateway architecture |
| **Steering/** | Standards, conventions, and session context for AI agent collaboration |
| **Servers/** | Individual MCP server details (one page per server) |
| **skills/** | Agent skills (e.g. the planned gateway-management skill) |

The [confluence-sync](./confluence-sync.md) page is the ref registry mapping local files to Confluence pages for this project.

## MCP Servers

| Server | Repo | Internal Doc |
| --- | --- | --- |
| **Datadog** | [datadog-mcp-server](https://bitbucket.org/tolunaengineering/datadog-mcp-server) | [Datadog MCP Server](./servers/datadog.md) |
| **Usain Bolt** | [usain-bolt-mcp-server-py](https://bitbucket.org/tolunaengineering/usain-bolt-mcp-server-py) | [Usain Bolt MCP Server](./servers/usain-bolt.md) |
| **ENV1** | [toluna-env1-mcp-server](https://bitbucket.org/tolunaengineering/toluna-env1-mcp-server) | [ENV1 MCP Server](./servers/env1.md) |
| **Chamberlain** | [chamberlain-mcp-server-py](https://bitbucket.org/tolunaengineering/chamberlain-mcp-server-py) | [Chamberlain MCP Server](./servers/chamberlain.md) |
| **Google Docs** | [toluna-google-docs-mcp-server](https://bitbucket.org/tolunaengineering/toluna-google-docs-mcp-server) | [Google Docs MCP Server](./servers/google-docs.md) |
| **Platform KB** (native connector; 4 KBs) | (native connector) | [Confluence Platform KB](./servers/confluence-kb.md) |

## Key Documentation

| Topic | Page |
| --- | --- |
| How to build an MCP server | [MCP Development Guide](./docs/mcp-development-guide.md) |
| Remote gateway architecture | [Remote MCP Gateway](./docs/remote-mcp-gateway.md) |
| Generic handler & V2 design | [MCP Gateway — Generic Handler & V2](./docs/mcp-gateway-v2-design.md) |
| Microservice promotion requirements | [MCP Gateway Microservice Requirements](./docs/mcp-gateway-microservice-requirements.md) |
| Headless / M2M auth design | [MCP Gateway — Headless / M2M Auth](./docs/mcp-gateway-m2m-headless-auth-design.md) |
| Public MCP Gateway design | [Public MCP Gateway — Design](./docs/public-mcp-gateway-design.md) |
| Design alternatives & incidents | [MCP Design Alternatives & Rejected Options](./docs/mcp-design-alternatives.md) |
| KB auto-sync | [KB Sync Scheduler](./docs/kb-sync-scheduler.md) |
| Agent behavioral testing | [Driftwatch Test Framework](./docs/driftwatch-test-framework.md) |
| Standards & conventions | [MCP Development Protocol](./steering/mcp-development-protocol.md) |
| Current state & collaboration | [Session Context](./steering/session-context.md) |
| Gateway operator runbook | [MCP Platform Operations](./steering/mcp-platform-operations.md) |
| Confluence sync protocol | [Confluence Sync](./steering/confluence-sync.md) |

## Contact

- **Platform Engineering** — gateway infrastructure and onboarding
- **Platform Community Channel** — MS Teams
