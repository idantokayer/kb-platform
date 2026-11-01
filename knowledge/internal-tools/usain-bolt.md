---
title: Usain Bolt - E2E Test Automation Platform
description: Toluna's E2E test automation platform — Playwright tests run against any environment, sharded across cloud containers, with real-time monitoring and Allure HTML reports.
tags: [plm, internal-tools, usain-bolt, testing, playwright, e2e]
---

# Usain Bolt - E2E Test Automation Platform

Part of [Internal Tools](./index.md).

## What it is

**Usain Bolt** is Toluna's end-to-end (E2E) test automation platform. It allows QA engineers, developers, and automated schedules to run Playwright-based browser tests against any Toluna environment, monitor their progress in real time, and review rich HTML test reports — all from a single unified dashboard.

The platform has two parts:

- **Usain** (backend) — The orchestration engine that queues tests, splits them into parallel shards, runs them on cloud containers, collects results, and generates reports.
- **Bolt Control** (frontend) — The web dashboard where users trigger tests, watch progress, filter results, and access reports.

## What problem it solves

- **Manual E2E testing doesn't scale**: Usain Bolt automates the entire lifecycle across dozens of microservices and multiple environments.
- **Slow test execution**: Usain automatically shards tests across parallel containers (up to 16 tests per shard), drastically reducing execution time.
- **No visibility into test health**: Bolt Control provides a real-time dashboard with status tracking, filtering, and historical reports.
- **Fragmented scheduling**: Integrates with AWS EventBridge for automated scheduled runs with full visibility.
- **No actionable reports**: Usain generates branded Allure HTML reports with screenshots, videos, trend analysis, and failure categorization.

## Key capabilities

- **Run tests on demand** — select a system/tribe, pick an environment, optionally filter by tags, click "Run Test".
- **Automated scheduled runs** — daily, hourly, or custom cron via AWS EventBridge, distinguished in the dashboard with a clock icon.
- **Real-time progress monitoring** — color-coded progress bars; status transitions Queued → Initializing → In Progress → Finishing → Done; cancel any running test.
- **Rich Allure reports** — executive summary, per-test results, screenshots/videos of failures, trend analysis, flakiness detection.
- **Retry failed tests** — replay button re-executes with the exact same parameters.
- **Flexible test filtering** — run subsets via Playwright tags (e.g. `@smoke`, `@regression`).
- **Multi-environment support** — see table below.
- **Notifications** — Microsoft Teams summary cards and SNS topics.
- **Admin analytics** — user engagement, conversion funnels, activity heatmaps, manual vs scheduled trends.

### Environments

| Environment | Purpose |
| --- | --- |
| PROD | Production monitoring and validation |
| STG | Pre-production staging (single-test lock enforced) |
| QAB | QA environment B |
| QAC | QA environment C |
| TRN | Training/development environment |

**Note:** STG has a concurrency lock — only one test can run at a time to ensure deterministic results.

## Who uses it

| Role | How they use it |
| --- | --- |
| QA Engineers | Run targeted suites, investigate failures, retry flaky tests, monitor regression health |
| Developers | Validate feature branches before merge, run smoke tests after deployments |
| Engineering Managers | Monitor environment health, track test reliability trends |
| Automation (scheduled) | Continuous regression testing on production and staging |
| DevOps / Platform | Administer the platform, manage system configs, review usage analytics |

## The Bolt Control dashboard

### Main reports table

A filterable, searchable table of all test runs. Columns: **System**, **Environment**, **Suite**, **Date**, **Status** (color-coded progress bar or chip), **Report** (link to Allure HTML). Additional columns (Branch, Tags, Scheduled indicator) toggle via a Jira-style column manager.

### Run Test modal

A guided form: select a system, choose the environment, optionally select tags/branch/advanced options (browser, parallelism, retries), execute.

### Search and filtering

Global search across system names, branches, suites, and error messages; combinable per-column filters that persist during the session.

## How a test execution works (simplified)

1. **You click "Run Test"** → request is queued
2. **Test is created** → unique record stored with all parameters
3. **Tests are discovered** → automation repo cloned; Playwright counts matching tests
4. **Tests are validated** → checked against the test management system
5. **Tests are sharded** → split into parallel batches (up to 16 per container)
6. **Containers run** → each shard runs in an isolated cloud container
7. **Results are collected** → each shard uploads results to cloud storage
8. **Report is generated** → merged into a single Allure HTML report
9. **Notifications are sent** → Teams messages and event notifications dispatched
10. **Dashboard updates** → status changes to "Done" with a link to the report

## Supported browsers

Chromium (default), Firefox, WebKit (Safari engine).

## Test configuration options

| Option | Description | Default |
| --- | --- | --- |
| Branch | Git branch of the automation repo to use | main |
| Browser | Which browser engine to run tests on | Chromium |
| Workers | Number of parallel workers per shard | 16 |
| Timeout | Max time per test (milliseconds) | 30000 |
| Retries | How many times to retry failed tests | 0 |
| Tags | Playwright test tags to filter execution | All tests |
| Screenshots | When to capture screenshots | Only on failure |
| Video | When to record video | Retain on failure |
| Trace | When to capture Playwright traces | Retain on failure |

## Get started

Everyone in the organization can run E2E tests at [https://bolt.toluna.me/](https://bolt.toluna.me/) — no special permissions required. Log in and start testing. Notifications flow to Microsoft Teams.

## Who maintains it

Owned and maintained by the **QA Automation and Platform** team.
