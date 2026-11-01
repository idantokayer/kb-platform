---
title: Confluence Platform KB
description: Bedrock Knowledge Bases backed by Confluence spaces, exposed as MCP tools via AgentCore Gateway — four KBs (public + internal), OAuth-based crawl, and the runbook to add another.
tags: [plm, mcp, server, knowledge-base, bedrock, confluence]
---

# Confluence Platform KB MCP

Bedrock Knowledge Bases backed by Confluence spaces, exposed as MCP tools via AgentCore Gateway.

Part of [MCP Development → Servers](../index.md). Auto-sync: [KB Sync Scheduler](../docs/kb-sync-scheduler.md).

## Knowledge Bases

| KB Name | KB ID | Data Source | Spaces | Gateway Target | Visibility |
| --- | --- | --- | --- | --- | --- |
| `knowledge-base-confluence-platform` | `WDWOAQWHUH` | `VZEA1KGFQO` | SOA, AUT | `confluence-platform-kb` | Public (Dev Platform Group) |
| `knowledge-base-platform-internal` | `IMLEVBMENE` | `W7OBNUOHSD` | PLM | `platform-internal-kb` | Platform team internal |
| `knowledge-base-devops-internal` | `JWDPJCK3HH` | `IH9X1QFFG5` | DCI | `devops-internal-kb` | DevOps team internal |
| `knowledge-base-test-automation-internal` | `ZIJLU8B5OA` | `7X2VSVKKSX` | ATP | `aut-internal-kb` | Test Automation team internal |

All KBs are **MANAGED** type, share the `confluence-kb-gateway-8dkmlwmoa3` gateway, and use the same `confluence_crawl_token` OAuth secret. Space-level filtering (`inclusionSpaceKeys`) controls what each KB indexes. All KBs are synced every 4 hours via the `kb-sync-scheduler` Lambda (see [KB Sync Scheduler](../docs/kb-sync-scheduler.md)).

## Tools

- `Retrieve` — Query the knowledge base and retrieve information
- `AgenticRetrieveStream` — Streaming endpoint for multi-step agentic retrieval with planning

## Architecture

Unlike other MCP servers, this uses a **native Bedrock KB connector** — no custom handler Lambda:

```
mcp.toluna.me/plm/plm-kb → proxy → confluence-kb-gateway-8dkmlwmoa3 → Bedrock KB connector → OpenSearch → Confluence content
```

Separate gateway because KB connectors are a different AgentCore target type.

## Why OAuth (not API tokens)

Toluna's Atlassian org **blocks API token access** for Confluence/Jira (Bitbucket exempt). The Bedrock connector uses OAuth 2.0 with rotating refresh tokens.

- Secret: `confluence_crawl_token` in Secrets Manager (shared by all KBs)
- Format: `{confluenceAppKey, confluenceAppSecret, confluenceAccessToken, confluenceRefreshToken, hostUrl}`
- Bedrock auto-refreshes tokens (needs `secretsmanager:PutSecretValue`)
- Initial token obtained via `scripts/get_confluence_oauth_token.py` (one-time browser consent)

## Creating Another KB from a Confluence Space

### Step 1: Create the Managed KB (via API — no console needed)

**1a. Create the execution role** (Bedrock assumes it to read the OAuth secret) with a trust policy for `bedrock.amazonaws.com` (scoped to account `457416912276` and `knowledge-base/*`), plus an inline policy granting `secretsmanager:GetSecretValue` + `PutSecretValue` on the `confluence_crawl_token` secret.

**1b. Create the MANAGED KB** via the raw `PUT /knowledgebases` API (the SDK skeleton only shows VECTOR type), with `knowledgeBaseConfiguration.type = "MANAGED"` and `managedKnowledgeBaseConfiguration.embeddingModelType = "MANAGED"`. Sign with SigV4 (service `bedrock`). Poll `get-knowledge-base` until `ACTIVE` (~1 minute).

### Step 2: Create the Data Source with Space Filter

Use the API (the console doesn't expose all filter options). The `connectorParameters` set `type: CONFLUENCE`, `authType: OAUTH2`, the `secretArn`, crawl flags (page/blog/attachments; no personal/archived spaces), and `filterConfiguration.inclusionSpaceKeys: ["YOUR_SPACE_KEY"]`. Apply via raw `PUT /knowledgebases/{KB_ID}/datasources`.

### Step 3: Trigger Sync

```shell
aws bedrock-agent start-ingestion-job --knowledge-base-id <KB_ID> --data-source-id <DS_ID> --profile datalab --region us-east-1
```

### Step 4: Expose via Gateway Target

Add a connector target on `confluence-kb-gateway-8dkmlwmoa3` (via SigV4 POST to `bedrock-agentcore-control`), configuring `AgenticRetrieveStream` + `Retrieve` for the KB ID and `GATEWAY_IAM_ROLE` credentials. Also add the new KB ARN to the gateway role policy (`bedrock:GetKnowledgeBase`, `bedrock:Retrieve` on the KB ARN; `bedrock:AgenticRetrieveStream` on `*`). Tools appear as `<target-name>___Retrieve` and `<target-name>___AgenticRetrieveStream`.

### Step 5: Add to Auto-Sync

Add the KB to the `kb-sync-scheduler` Lambda's `KBS` list. See [KB Sync Scheduler](../docs/kb-sync-scheduler.md).

## Important Notes

- **OAuth secret is shared** — all Confluence KBs use `confluence_crawl_token`. The token reads all spaces the user can see; `inclusionSpaceKeys` controls what gets indexed.
- **Console doesn't show filters** — verify via `get-data-source` API.
- **Space KEY vs NAME** — use the short key (`SOA`, `PLM`, `DCI`, `AUT`, `ATP`), not the display name.
- **One data source per KB** — pointing to one or more spaces.

## Existing Gateway Targets

| Target Name | Target ID | KB | Spaces |
| --- | --- | --- | --- |
| `confluence-platform-kb` | `YT5VJLB7PA` | `WDWOAQWHUH` | SOA, AUT |
| `platform-internal-kb` | `NRFHUQ8INL` | `IMLEVBMENE` | PLM |
| `devops-internal-kb` | `0VJ4KIBHTE` | `JWDPJCK3HH` | DCI |
| `aut-internal-kb` | `HDDIEPKYUU` | `ZIJLU8B5OA` | ATP |

## Troubleshooting

| Issue | Fix |
| --- | --- |
| Stale results | Re-sync the data source (or wait for the 4h scheduler) |
| OAuth expired | Run `get_confluence_oauth_token.py` for fresh tokens |
| Failed documents | Usually large pages or unsupported macros — safe to ignore |
| Filter not visible in console | Expected — verify via `get-data-source` API |
