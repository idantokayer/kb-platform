---
title: Radio - Lambda Power Tuning
description: Toluna's self-hosted AWS Lambda Power Tuning tool with a web UI — finds the optimal memory/power config for a Lambda by running it at multiple settings and charting cost vs speed.
tags: [plm, internal-tools, radio, lambda, cost-optimization, performance]
---

# Radio - Lambda Power Tuning

Part of [Internal Tools](./index.md).

## What it is

**Radio** is Toluna's self-hosted deployment of the [AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning) tool with a simple web UI. It helps developers find the optimal memory/power configuration for their Lambda functions — balancing cost and performance based on real execution data, not guesswork.

Instead of manually adjusting Lambda memory settings and hoping for the best, Radio runs your function multiple times at different memory configurations (from 128 MB to 10 GB) and produces a clear visualization showing you the sweet spot between cost and speed.

**Try it now:** [https://radio.toluna.me/](https://radio.toluna.me/) (VPN required)

## What problem it solves

- **Over-provisioned Lambdas waste money**: Radio shows you exactly where the cost-performance curve flattens so you can right-size.
- **Under-provisioned Lambdas are slow**: Lambda CPU scales linearly with memory. Radio reveals this with real data.
- **No way to test without trial and error**: Radio automates the entire process in a single click.
- **Barrier to entry**: Radio wraps the underlying Step Functions-based tool (CLI, IAM, state machines) in a simple web form.

## How it works

1. You provide the **ARN** of the Lambda function you want to optimize
2. You choose an **optimization strategy** (cost, speed, or balanced)
3. You select which **memory configurations** to test (e.g., 128, 256, 512, 1024, 1536, 3008 MB)
4. You set how many times to **invoke** the function per configuration (more = more accurate)
5. Optionally, you provide a **JSON payload** (the event your Lambda normally receives)
6. You click **"Start power tuner"**
7. The tool invokes your Lambda multiple times at each memory setting and collects execution metrics
8. After a few seconds to minutes, you get a **visualization chart** and clear recommendations

## What you get back

After tuning completes, Radio displays:

- **Recommended Memory** — The optimal memory configuration based on your chosen strategy
- **Average Cost** — The cost per invocation at the recommended setting
- **Average Duration** — The execution time at the recommended setting
- **Interactive Chart** — Cost and duration across all tested memory values

## How to use it

### Step 1: Get your Lambda ARN

```
arn:aws:lambda:us-east-1:123456789012:function:my-function-name
```

Find this in the AWS Console under Lambda → your function → "Function ARN".

### Step 2: Open Radio

1. Connect to **VPN**
2. Navigate to [https://radio.toluna.me/](https://radio.toluna.me/)

### Step 3: Configure the tuning run

| Setting | What to choose | Recommendation |
| --- | --- | --- |
| Operation | "New Tuner" or "Existing execution ID" | Use "New Tuner" for fresh analysis |
| Lambda ARN | Paste your function's ARN | Required |
| Strategy | Cost, Speed, or Balanced | Start with "Balanced" for most use cases |
| Invocation Number | How many times to invoke per memory value | 10-50 for quick results, 100+ for accuracy |
| Power Values | Which memory configs to test | "Default" (128-3008) for initial exploration |
| Execute in Parallel | Run all memory configs simultaneously | Yes (faster results) |
| JSON Payload | The event your Lambda normally receives | Include it for realistic results |

### Step 4: Run and interpret results

The chart shows Memory (MB) on the X-axis, Cost and duration on the Y-axis. The point where the curve flattens is your optimal configuration.

### Step 5: Apply the recommendation

Update your Lambda's memory configuration in Terraform (or the AWS Console) to the recommended value.

## Optimization strategies explained

| Strategy | What it optimizes for | When to use |
| --- | --- | --- |
| **Cost** | Lowest price per invocation | Background jobs, batch processing, anything not latency-sensitive |
| **Speed** | Fastest execution time | API endpoints, user-facing Lambdas where latency matters |
| **Balanced** | Best tradeoff between cost and speed (configurable weight 0-1) | Most use cases — good starting point |

The "Balanced" strategy lets you set a weight between 0 (pure cost) and 1 (pure speed). A weight of 0.5 gives equal importance to both.

## Power value presets

| Preset | Values (MB) | Use case |
| --- | --- | --- |
| Default | 128, 256, 512, 1024, 1536, 3008 | Quick exploration across the full range |
| Moderate | 512, 1024, 1536, 2048, 4096 | Mid-to-high range for compute-heavy functions |
| Custom | You choose | Fine-tuning after initial exploration |
| All possible values | Every 64 MB from 128 to 10240 | Maximum precision (takes longer) |

## Payload templates

Radio includes built-in payload templates for common Lambda triggers: **API Gateway**, **SQS**, **SNS**, and **None** (empty payload). Select a template and modify it with your real data.

## Tips for accurate results

- **Use a realistic payload** — the function behaves differently with different inputs.
- **Run enough invocations** — at least 10 per configuration to reduce noise from cold starts and variance.
- **Be aware of side effects** — the tool actually invokes your Lambda. Use a test/dev version if it writes to a database or sends emails.
- **Start broad, then narrow** — first run "Default" values, then "Custom" values around the optimal range.
- **Check the Datadog link on errors** — Radio provides a direct link to the Step Functions execution logs in Datadog for debugging.

## Re-fetching previous results

Every tuning run produces an **execution ID**. To re-fetch: switch the operation dropdown to "Existing execution ID", paste the execution ID, and click "Fetch tuner result". The execution ID is also saved in your browser's local storage.

## Common troubleshooting

| Issue | Cause | Fix |
| --- | --- | --- |
| Tuning fails immediately | Invalid Lambda ARN | Double-check the ARN format and region |
| Timeout errors | Low memory values cause function to time out | Start from a higher memory value (e.g., 512 MB) |
| "Parameter not found" | Missing or incorrect payload | Provide the correct event payload your function expects |
| Results take too long | Too many invocations or "All possible values" | Reduce invocation count or use fewer power values |
| Cannot access the page | Not on VPN | Connect to corporate VPN first |

## Who maintains it

Owned and maintained by the **Platform team**. Based on the open-source [AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning) project by Alex Casalboni, with a UI layer from [aws-power-tuner-ui](https://github.com/mattymoomoo/aws-power-tuner-ui).
