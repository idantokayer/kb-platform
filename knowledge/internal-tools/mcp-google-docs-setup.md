---
title: Google Docs MCP — Setup Guide
description: Connect your AI agent to Toluna's Google Workspace for read-only access to documents, spreadsheets, and search across shared drives.
tags: [plm, internal-tools, mcp, google-docs, setup]
---

# Google Docs MCP — Setup Guide

Connect your AI agent to Toluna's Google Workspace for reading documents, spreadsheets, and searching across shared drives.

Part of [MCP Remote Tools](./mcp-remote-tools.md).

**Package:** `@toluna-tools/toluna-google-docs-mcp-server` (internal, CodeArtifact)

**Access:** Read-only. Searches and reads documents accessible to your @toluna.com Google account.

## Quick Start

### 1. Prerequisites

- Node.js 18+
- AWS CLI configured (for CodeArtifact authentication)
- Toluna Google Workspace account (@toluna.com)

### 2. Install from CodeArtifact

```shell
# Authenticate with CodeArtifact (valid for 12 hours)
aws codeartifact login --tool npm --repository npm-shared --domain toluna-artifact-repository --domain-owner 014931512072 --region us-east-1

# Install
npm install -g @toluna-tools/toluna-google-docs-mcp-server
```

### 3. Add to your agent config

```json
"google-docs": {
  "command": "toluna-google-docs-mcp-server",
  "args": [],
  "disabled": false,
  "env": {}
}
```

### 4. Authenticate

On first use, a browser window opens for Google OAuth. Log in with your **@toluna.com** account and grant read access. The token is stored at `~/.kiro/mcp-tokens/google-docs/` and refreshes automatically.

## Available Tools

| Tool | Description |
| --- | --- |
| `read_google_doc` | Read content from a Google Doc by URL or ID |
| `read_google_sheet` | Read data from a Google Sheet (with optional range) |
| `search_google_docs` | Smart search with relevance ranking (title match priority) |
| `advanced_search_google_docs` | Search with date filters, sort order, and limits |
| `list_recent_docs` | List recently modified documents |

## Usage Examples

**Read a document:**

```
read_google_doc(doc_url_or_id: "https://docs.google.com/document/d/1ABC...")
```

**Search for documents:**

```
search_google_docs(query: "architecture diagram", limit: 5)
```

**Read a specific sheet range:**

```
read_google_sheet(sheet_url_or_id: "https://docs.google.com/spreadsheets/d/1XYZ...", range: "Sheet1!A1:D50")
```

## Notes

- **Read-only** — cannot create or modify documents
- **Toluna accounts only** — requires @toluna.com Google Workspace login
- **Shared drives supported** — searches across personal and shared drives you have access to
- **Runs locally** — not on the remote gateway (needs your Google OAuth token)
- **Token location:** `~/.kiro/mcp-tokens/google-docs/`

## Troubleshooting

| Issue | Fix |
| --- | --- |
| "Not authorized" errors | Re-authenticate: delete `~/.kiro/mcp-tokens/google-docs/` and restart |
| No results from search | Verify the documents are accessible to your @toluna.com account |
| Install fails | Ensure CodeArtifact token is fresh: re-run `aws codeartifact login ...` |
| Command not found | Check install: `npm list -g @toluna-tools/toluna-google-docs-mcp-server` |

## Upgrading

```shell
# If upgrading from v0.2.x (old package name):
npm uninstall -g @toluna-tools/toluna-google-doc-mcp-server
npm install -g @toluna-tools/toluna-google-docs-mcp-server
```

Note the package name changed from `doc` (singular) to `docs` (plural) in v0.3.0.

## Support

Platform Community Channel on Teams. Repo: [tolunaengineering/toluna-google-docs-mcp-server](https://bitbucket.org/tolunaengineering/toluna-google-docs-mcp-server)
