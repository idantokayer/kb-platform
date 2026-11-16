---
title: RTFM - Authentication
description: The three credential layers in RTFM — Cognito browser login, the internal rotated proxy token, and per-space git credentials — and how each is stored and rotated.
tags: [plm, internal-tools, rtfm, authentication, cognito, security]
---

# RTFM - Authentication

Part of [RTFM](./index.md).

RTFM keeps three credential layers separate.

| Layer | Protects | Mechanism |
|---|---|---|
| Front door | browser access to `/<space>/api/*` | Cognito OIDC (Hosted UI, code flow) + JWT |
| Internal proxy token | proxy Lambda to microVM | `X-aws-proxy-auth`, short-lived, rotated |
| Git credential | microVM to Bitbucket | short-lived access token injected at `/run` |

> Scope: authentication only. Per-space `read_groups` authorization is a later phase.

## Cognito (front door)

- **One user pool per account**, named `rtfm-<env_type>-users`, with Hosted UI domain `rtfm-<account_id>`. It is shared by every env in the account (`qa`, `stg`, `dev`, `idan`).
- The single app client lists **every env's** OIDC callback and logout URL, derived from env1 (`app-env.json` env list × `shared-env.json` domain). The callback path is `/<space>/oauth2/callback`.
- The pool is **admin-create-only**. Create a user with:

```bash
aws cognito-idp admin-create-user --user-pool-id <pool-id> \
  --username <email> \
  --user-attributes Name=email,Value=<email> Name=email_verified,Value=true \
  --profile rtfm-non-prod
```

- The **shared** layer creates Cognito. The **app** layer looks the pool and client up by name (`data.aws_cognito_user_pool*`), avoiding a brittle remote-state dependency.
- `/oauth-config` is served by the proxy from environment variables and bootstraps the SPA's OIDC flow without a microVM call.
- The WebSocket `$connect` authorizer validates a Cognito JWT passed as `?token=`.
- In read-only mode, GitHub sign-in and token entry are allowed while content changes stay blocked.

MCP is on the public route because MCP clients cannot complete the browser flow — see the caveat in [Architecture](./architecture.md).

## Internal proxy token

Minted by LifecycleManager at launch, stored in the routes table, injected by the proxy, and refreshed by TokenRotation. See [Control Plane](./control-plane.md). It never reaches the browser.

## Git credentials (per space)

The microVM clones and pulls over **HTTPS**. LifecycleManager chooses the first configured backend:

1. **Bitbucket OAuth 2.0 (`client_credentials`)** — primary. LifecycleManager exchanges a Bitbucket OAuth consumer (client id/secret in SSM) for an access token valid about 2 hours, and sends it in `/run` as `gitToken` with `gitTokenHost=bitbucket.org` and `gitAuthScheme=x-token-auth`. The client secret never enters the VM. The consumer needs **Repositories: Read** only.
2. **GitHub fine-grained PAT** — long-lived token in SSM, used as `https://x-access-token:<pat>@github.com`.
3. **SSH key** — dormant fallback; a read-only deploy key plus known-hosts, mutually exclusive with the token path.

Inside the VM, `entrypoint.sh` writes the token to a `0600` file under `/tmp/ok-run/`, configures `credential.helper store`, and rewrites SSH-form remotes to HTTPS with `url.<host>.insteadOf`. `/api/sync/status` may report `"transport":"ssh"` for SSH-form remote URLs; this is cosmetic and pulls still succeed over HTTPS.

### SSM parameters (shared layer)

| Parameter | Purpose |
|---|---|
| `/rtfm/bb_at` | Bitbucket token; Terraform creates a placeholder with `ignore_changes`, the real value is seeded out-of-band |
| `/rtfm/spaces/github-pat` | GitHub PAT fallback |
| `/rtfm/spaces/ssh-key` | SSH key fallback |

Seed or rotate the Bitbucket token (do not paste real values into tickets or chat):

```bash
aws ssm put-parameter --name /rtfm/bb_at --type SecureString --overwrite \
  --value "<TOKEN>" --profile rtfm-non-prod
```

Changing a value needs no image rebuild; relaunch the space.

### Security practices

- Credentials live only in SSM SecureString — never in code, DynamoDB or logs.
- **Rotate immediately** if a token is ever exposed, then re-seed SSM in **both** accounts.
