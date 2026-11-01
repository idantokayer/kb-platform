---
title: pipeline-controller 3.1.7 — Release Notes
description: Release notes for pipeline-controller 3.1.7 — adds pipeline and deployment cleanup when a PR is declined or closed.
tags: [plm, terraform, module, pipeline-controller, release-notes]
---

# 3.1.7

Release notes for the [pipeline-controller](./pipeline-controller.md) module.

| Field | Value |
| --- | --- |
| **Released** | 2026-07-28 |
| **Released By** | Eli Meitner |
| **PR** | [PLM-3947: cleanup pipeline and deployments on PR declined/closed](https://bitbucket.org/tolunaengineering/terraform-aws-pipeline-controller/pull-requests/39) |
| **Ticket** | [PLM-3947](https://toluna.atlassian.net/browse/PLM-3947) |
| **Branch** | `master` |
| **Tag** | `3.1.7` |

---

## What Changed

- Added `PULL_REQUEST_CLOSED` to CodeBuild webhook event filter — the source build now triggers when a PR is declined or superseded
- Added PR state check (DECLINED/SUPERSEDED) `codebuild-source-pr` — detects closed PRs early and initiates cleanup
- Refactored draft + declined cleanup into a shared `cleanup_and_exit()` function — eliminates code duplication between the draft-skip and declined-cleanup paths
- Single Bitbucket API call now fetches both `draft` and `state` fields — reduces API calls and simplifies the logic

**Problem solved:** Previously, declining a PR left the pipeline running until the CodeDeploy lifecycle hook timed out (~1 hour), blocking other PRs from deploying to the same environment.

**Files changed:** `modules/bg_controller/main.tf`, `modules/bg_controller/templates/legacy/pr-created-buildspec-source.yml.tpl`, `modules/bg_controller/templates/cd/pr-created-buildspec-source.yml.tpl`, `CHANGELOG.md`, `README.md`

---

## Migration Steps

No migration required — drop-in upgrade. The new webhook filter and buildspec logic are additive and backward-compatible.

---

## Affected Services

All services using pipeline-controller can upgrade at their next deployment. No action required — existing services will simply gain the declined-PR cleanup behavior when they bump to 3.1.7.
