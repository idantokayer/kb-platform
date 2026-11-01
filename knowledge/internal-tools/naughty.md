---
title: Naughty - Teams Notification Service
description: Toluna's internal notification microservice that sends rich Adaptive Card messages to Microsoft Teams via Workflows, using S3-stored JSON templates and a single API endpoint.
tags: [plm, internal-tools, naughty, notifications, teams, adaptive-cards]
---

# Naughty - Teams Notification Service

Part of [Internal Tools](./index.md).

## What it is

**Naughty** is Toluna's internal notification microservice that sends rich [Adaptive Card](https://adaptivecards.io/) messages to Microsoft Teams channels. It replaces the deprecated Office 365 Connectors (webhooks) with a modern, template-based approach powered by Microsoft Teams Workflows.

Instead of each service maintaining its own webhook integration, Naughty provides a single API endpoint that any service, pipeline, or automation can call to deliver beautifully formatted notifications to Teams.

## What problem it solves

- **Office 365 Connectors are deprecated**: Naughty is the replacement — using Teams Workflows and Adaptive Cards.
- **Duplicated webhook logic everywhere**: Naughty centralizes notification logic into a single API call.
- **Ugly plain-text messages**: Naughty renders rich Adaptive Cards with colors, icons, facts tables, conditional formatting, and branding.
- **Hard to maintain notification templates**: Templates are JSON files stored in S3 — update once, every sender gets the new design.

## How it works

1. You send a **POST request** to Naughty's API with your notification data
2. Naughty publishes the message to an **SNS topic**
3. An **SQS queue** picks up the message and triggers the Naughty Lambda
4. The Lambda loads the matching **Adaptive Card template** from S3 (based on `NOTIFICATION_TYPE`)
5. A **card designer Lambda** (Node.js) renders the template with your data (variable substitution + conditional logic)
6. The final Adaptive Card is **POSTed to the Teams Workflow URL** (stored in SSM or passed directly)
7. The message appears in your Teams channel as a rich, formatted card

## How to use it

### Sending a notification (the simple version)

```shell
curl --location 'https://naughty.toluna.me/send_teams_message' \
--header 'Content-Type: application/json' \
--data '{
    "NOTIFICATION_TYPE": "general",
    "HOOK_ID": "/infra/my-team/teams_hook",
    "TITLE": "Deployment Complete",
    "MESSAGE": "Service X deployed successfully to production"
}'
```

### Understanding the key parameters

| Parameter | Required | Description |
| --- | --- | --- |
| `NOTIFICATION_TYPE` | Yes | The name of your Adaptive Card template file in S3 (without `.json`) |
| `HOOK_ID` | Yes | Either an SSM parameter path containing the Teams Workflow URL, OR a direct `https://` Workflow URL |
| _(your fields)_ | Depends on template | Any additional fields your template expects (e.g., `TITLE`, `MESSAGE`, `STATUS`) |

### Available template examples

| Template name | Use case | Expected fields |
| --- | --- | --- |
| `general` | Simple title + message notification | `TITLE`, `MESSAGE` |
| `alfred_deployment` | Deployment status with details | `TITLE`, `MESSAGE`, `DEPLOYMENT_STATUS`, `ENV_NAME`, `SERVICE_NAME`, `IMAGE_TAG`, etc. |
| `jenkins_deployment` | Jenkins job status notification | `MESSAGE`, `JOB_STATUS`, `JOB_URL` |
| `ami_retention` | AMI cleanup Lambda results | `MESSAGE`, `STATUS` |

### Step-by-step: Adding a new notification for your service

#### 1. Design your Adaptive Card

Use the [Adaptive Cards Designer](https://adaptivecards.io/designer/) to visually create your card template (select **Microsoft Teams** as the host app). Use `${VARIABLE_NAME}` placeholders for dynamic content:

```json
{
    "type": "message",
    "body": [
        {
            "type": "TextBlock",
            "weight": "Bolder",
            "text": "${TITLE}"
        },
        {
            "type": "TextBlock",
            "text": "${MESSAGE}",
            "wrap": true,
            "color": "condition: '${STATUS}' == 'success' and 'Good' or 'Attention'"
        }
    ]
}
```

#### 2. Upload your template to S3

Upload the JSON file to the S3 bucket: `naughty-adaptive-cards-prod-data`. The filename (without `.json`) becomes your `NOTIFICATION_TYPE`.

#### 3. Set up a Teams Workflow URL

1. Open Teams → click **Workflows**
2. Choose **"Post to a channel when a webhook request is received"** template
3. Select your target channel
4. Copy the generated Workflow URL
5. Edit the flow → open "Send each adaptive card" → change "Post as" to **"User"**

Store the URL in an SSM parameter (e.g., `/infra/my-team/teams_hook`) or pass it directly in `HOOK_ID`.

#### 4. Send your first notification

```shell
curl --location 'https://naughty.toluna.me/send_teams_message' \
--header 'Content-Type: application/json' \
--data '{
    "NOTIFICATION_TYPE": "my_service_deploy",
    "HOOK_ID": "/infra/my-team/teams_hook",
    "TITLE": "My Service Deployed",
    "MESSAGE": "Version 2.1.0 deployed to production",
    "STATUS": "success"
}'
```

### Multiple channels

Store multiple Workflow URLs in a single SSM parameter as a **comma-separated list**. Naughty sends to all of them:

```
/infra/my-team/teams_hook = "https://workflow1.url,https://workflow2.url"
```

### Conditional formatting

Naughty supports conditional logic in templates for dynamic colors and visibility:

```
"color": "condition: '${STATUS}' == 'success' and 'Good' or '${STATUS}' == 'failure' and 'Attention' or 'Default'"

"isVisible": "condition: '${STATUS}' == 'failure'"
```

## Who's already using it

- **CI/CD Pipelines** — Deployment status notifications (success/failure/in-progress)
- **Usain Bolt** — E2E test result summaries (see [Usain Bolt](./usain-bolt.md))
- **Alfred** — Deployment orchestration status
- **AMI Retention Lambda** — Cleanup job results
- **Jenkins** — Job status notifications

## Troubleshooting

| Issue | Cause | Fix |
| --- | --- | --- |
| Card not arriving in Teams | Wrong Workflow URL or SSM path | Verify `HOOK_ID` SSM parameter exists and contains a valid URL |
| "No such file" error | Template not in S3 | Upload your `.json` template to `naughty-adaptive-cards-prod-data` |
| Card renders with raw `${}` placeholders | Missing fields in request | Ensure your POST body includes all fields the template expects |
| Conditional colors not working | Incorrect condition syntax | Use: `"condition: '${FIELD}' == 'value' and 'Color' or 'Default'"` |
| Card appears but "Post as" shows app | Workflow not configured | Edit the Teams Workflow → change "Post as" to "User" |

## Who maintains it

Owned and maintained by the **DevOps** team.
