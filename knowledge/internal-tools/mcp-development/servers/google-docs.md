---
title: Google Docs MCP Server
description: The Google Docs MCP server — local-only (stdio) TypeScript server for read access to Google Docs and Sheets via per-user OAuth; not on the remote gateway.
tags: [plm, mcp, server, google-docs, local]
---

# Google Docs MCP Server

Part of [MCP Development → Servers](../index.md). End-user setup: [Google Docs MCP — Setup Guide](../../mcp-google-docs-setup.md).

| Field | Value |
| --- | --- |
| **Repo** | [toluna-google-docs-mcp-server](https://bitbucket.org/tolunaengineering/toluna-google-docs-mcp-server) |
| **Package** | `@toluna-tools/toluna-google-docs-mcp-server` (npm) v0.2.1 |
| **Runtime** | Local only (stdio) |
| **Language** | TypeScript |
| **Profile** | N/A (not on remote gateway) |

## Tools

- `read_google_doc` — Read content from a Google Doc
- `read_google_sheet` — Read data from a Google Sheet (with optional range)
- `search_google_docs` — Search with smart relevance ranking
- `advanced_search_google_docs` — Search with sorting and date filters
- `list_recent_docs` — Recently modified documents

## Why Local Only

Google APIs require per-user OAuth tokens with Google-specific scopes. Each user must consent individually. The token is stored locally at `~/.mcp-auth/toluna-google-docs/`.

**Could it go remote?** Possibly with a domain-wide delegation service account — but that grants access to ALL docs of ALL users, which is a security concern. The per-user local approach is more appropriate.

## Installation

```shell
aws codeartifact login --tool npm --repository npm-shared --domain toluna-artifact-repository --domain-owner 014931512072 --region us-east-1
sudo npm install -g @toluna-tools/toluna-google-docs-mcp-server
```

## Operational Notes

- Requires `@toluna.com` Google account
- Searches personal AND shared drives (including Team Drives)
- Token refresh handled automatically
- Only TypeScript MCP server (predates the Python migration)
- Uses npm CodeArtifact registry (not PyPI)
