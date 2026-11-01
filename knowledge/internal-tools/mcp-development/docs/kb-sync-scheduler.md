---
title: KB Sync Scheduler
description: An automated Lambda that periodically re-ingests all Confluence-backed Bedrock Knowledge Bases every 4 hours, skipping any KB whose ingestion job is already running.
tags: [plm, mcp, knowledge-base, bedrock, lambda, scheduler]
---

# KB Sync Scheduler

Automated periodic sync for all Confluence-backed Bedrock Knowledge Bases. Prevents KBs from going stale when Confluence content changes.

Part of [MCP Development](../index.md). Related: [Confluence Platform KB](../servers/confluence-kb.md).

## Overview

| Field | Value |
| --- | --- |
| **Lambda** | `kb-sync-scheduler` |
| **Schedule** | Every 4 hours (`rate(4 hours)`) |
| **EventBridge Rule** | `kb-sync-every-4h` |
| **IAM Role** | `kb-sync-scheduler-role` |
| **AWS Account** | 457416912276 (datalab) |
| **Region** | us-east-1 |

## How It Works

```
EventBridge Schedule (every 4h)
  → Lambda: kb-sync-scheduler
    → For each KB in the list:
        - ListIngestionJobs -> check for active job
        - If IN_PROGRESS/STARTING -> skip (avoid duplicate)
        - If idle -> StartIngestionJob
    → Returns results per KB
```

The "check before start" logic respects Bedrock's quota of **1 concurrent job per KB** — if a sync is already running (manual or previous schedule), it skips rather than erroring.

## Configured KBs

The KB list is **hardcoded** in the Lambda (`KBS` array). Current entries:

```python
KBS = [
    {"name": "confluence-platform", "kb_id": "WDWOAQWHUH", "ds_id": "VZEA1KGFQO"},
    {"name": "platform-internal", "kb_id": "IMLEVBMENE", "ds_id": "W7OBNUOHSD"},
    {"name": "devops-internal", "kb_id": "JWDPJCK3HH", "ds_id": "IH9X1QFFG5"},
    {"name": "test-automation-internal", "kb_id": "ZIJLU8B5OA", "ds_id": "7X2VSVKKSX"},
]
```

## Adding a New KB to the Scheduler

1. Add an entry to the `KBS` array in `lambda_function.py`:
   ```python
   {"name": "<short-name>", "kb_id": "<KB_ID>", "ds_id": "<DS_ID>"},
   ```
2. Redeploy the Lambda:
   ```shell
   cd /tmp && zip -j kb-sync-scheduler.zip kb-sync-lambda.py
   aws lambda update-function-code \
     --function-name kb-sync-scheduler \
     --zip-file fileb://kb-sync-scheduler.zip \
     --profile datalab --region us-east-1
   ```
3. The Lambda's IAM role uses a wildcard (`knowledge-base/*`), so no permission change needed.

## IAM Role Permissions

`kb-sync-scheduler-role`:

- `AWSLambdaBasicExecutionRole` (managed) — CloudWatch logs
- Inline `BedrockKBSync`:
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["bedrock:StartIngestionJob", "bedrock:ListIngestionJobs"],
      "Resource": "arn:aws:bedrock:us-east-1:457416912276:knowledge-base/*"
    }]
  }
  ```

## Manual Trigger (Test)

```shell
aws lambda invoke \
  --function-name kb-sync-scheduler \
  --payload '{}' \
  --profile datalab --region us-east-1 \
  /tmp/response.json
cat /tmp/response.json
```

Expected — `started` (with job_id) or `skipped` (already running) per KB.

## Monitoring

- **CloudWatch Logs:** `/aws/lambda/kb-sync-scheduler`
- **Sync history:** `list-ingestion-jobs` per KB/DS
- No SNS alerting yet — failures are logged only.

## Design Notes

This is a **simple, focused implementation** — one Lambda with a hardcoded KB list, one EventBridge schedule. It intentionally avoids a heavier registry-driven / Step Functions design.

### Future Enhancements

- DynamoDB registry (add KBs without code changes)
- API Gateway endpoint for agent-triggered on-demand sync
- S3 event triggers for agent-produced content KBs
- SNS failure notifications
- Status dashboard
