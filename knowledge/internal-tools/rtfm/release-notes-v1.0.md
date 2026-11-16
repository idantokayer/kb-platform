---
title: RTFM - Release Notes v1.0
description: What shipped in RTFM v1.0 — edge and routing, env1-driven spaces, authentication, control plane, microVM data plane, infrastructure, and known limitations.
tags: [plm, internal-tools, rtfm, release-notes]
---

# RTFM - Release Notes v1.0

Part of [RTFM](./index.md).

## Edge and routing

- Private API Gateway REST API front door through the shared VPC endpoint, with a private custom domain managed by `switch-traffic`.
- WebSocket API for collab (`$connect` authorizer, relay Lambda, `ws-connections` table); collab itself is disabled.
- The proxy Lambda resolves `space -> live microVM` per request from `rtfm-<data-workspace>-routes`, caches immutable `/assets/*`, retries transient microVM 429s, and injects `X-aws-proxy-auth`, a loopback `Origin` and `X-OK-Forwarded-Host`.
- Base-path support throughout (`server.basePath`, for example `/platform`): web `fetch('/api/...')` calls and the agent-thread WebSocket URL respect the prefix.

## Space configuration

- Spaces are controlled from env1 (`app-env.json`, `<env>.spaces`), not tfvars. `space_image_arn` and `data_workspace` are env1-driven per env too.
- Multiple app envs can share one data workspace; records are tagged with the env (`<env>#<space>` keys, `/spaces/<env>/<space>` EFS paths).
- Adding, removing or re-pointing a space is a config change picked up on the next deploy or reconcile.

## Authentication

- One Cognito user pool per account for browser OIDC login (Hosted UI, code flow); the JWT authorizer gates `/<space>/api/*`.
- Two token layers kept separate: the front-door Cognito JWT, and the internal rotated `X-aws-proxy-auth` token.
- GitHub sign-in is allowed in read-only mode while content changes stay blocked.
- Scope: authentication only. Per-space `read_groups` authorization is a later phase.

## Control plane

- LifecycleManager (EventBridge reconcile) launches one microVM per space from the `rtfm` image via the `/run` hook, mints the proxy token and writes the route.
- TokenRotation refreshes tokens before they expire.
- Per-space `/run` config is honored: the VM clones the requested repo, branch and external URL.

## microVM data plane

- Container deployment with persistent content and read-only mode on by default.
- Private content repositories via the `/run` hook (read-only SSH deploy key plus known-hosts, never logged) or Bitbucket OAuth tokens.
- Read-only servers reject mutating API requests while still allowing GitHub credential setup; Docker source builds and web-editor startup are fixed.

## Infrastructure

- Three Terraform layers: shared (Cognito, SSM, image build prerequisites), data (DynamoDB, EFS), app (REST and WebSocket APIs, five Lambdas, access points, seed items, schedules). See [Terraform Layers](./terraform-layers.md).
- Encrypted EFS with mount targets, access points and a dedicated NFS security group, reusing the existing VPC and subnets.
- Shared resources can be destroyed and recreated with generated DNS targets while preserving the certificate and network infrastructure. Recreating EFS yields an empty filesystem with new identifiers.

## Editor, history and agent behavior

- History and agent-change diffs render images, callouts and other components.
- Recreated pages stay open and keep their comments when earlier removals finish late.
- Agent edits during a Git branch switch wait up to 20 s; an edit dropped by the switch returns a `branch-switched` error instead of a false success.
- Terminal text sent to an agent renders control characters as labels so they cannot end a session or act as keystrokes.

## Known limitations

- **Live collaboration is disabled.** Documents render read-only over HTTP; users refresh (or a 60 s poll) to see new content.
