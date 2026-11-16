---
title: RTFM - Deployment and Runbook
description: Step-by-step deploy order for RTFM per account/env, the image build and promotion procedure, DNS prerequisites, and the current deployment state of qa and prod.
tags: [plm, internal-tools, rtfm, deployment, runbook, dns]
---

# RTFM - Deployment and Runbook

Part of [RTFM](./index.md).

> Production changes require explicit approval. Never apply to prod without it.

## Prerequisites per account

- A public **ACM certificate** for the domain (issued).
- The shared **execute-api VPC endpoint** (lives in the net account).
- The private hosted zone **associated with the shared VPC** the corporate resolver forwards into (see [DNS](#dns)).
- AWS SSO profiles `rtfm-non-prod` / `rtfm-prod`; env1 config writes need `platform-net`.

## Deploy order (from scratch, per env)

1. **Build the microVM image** (shared bucket and role first):

   ```bash
   terraform -chdir=shared workspace select shared-non-prod
   terraform -chdir=shared apply \
     -target=aws_s3_bucket.microvm_artifacts \
     -target=aws_iam_role.microvm_build -target=aws_iam_role_policy.microvm_build
   # stage service/ (exclude node_modules, dist, .turbo, .git, docs),
   # copy docker/microvm/Dockerfile to the zip root, zip, upload:
   aws s3 cp rtfm-microvm.zip s3://rtfm-non-prod-microvm-artifacts/ --profile rtfm-non-prod
   aws lambda-microvms create-microvm-image --name rtfm \
     --code-artifact uri=s3://rtfm-non-prod-microvm-artifacts/rtfm-microvm.zip \
     --base-image-arn arn:aws:lambda:us-east-1:aws:microvm-image:al2023-1 \
     --build-role-arn <microvm_build_role_arn> \
     --cpu-configurations architecture=ARM_64 \
     --resources minimumMemoryInMiB=4096 \
     --additional-os-capabilities '["ALL"]' \
     --hooks 'port=9000,microvmHooks={run=ENABLED,runTimeoutInSeconds=60},microvmImageHooks={ready=ENABLED,readyTimeoutInSeconds=300}' \
     --profile rtfm-non-prod
   aws lambda-microvms get-microvm-image --image-identifier <arn> --query state
   ```

   - `--additional-os-capabilities '["ALL"]'` is **required** for EFS mounting.
   - Rebuilds use `update-microvm-image` with PUT semantics: re-pass **all** flags.
   - Version numbers auto-increment and are not selectable. LifecycleManager always launches the latest active version.

2. **shared** — `terraform apply` in `shared-non-prod` (SSM params, Cognito, build prerequisites). Set `bitbucket_token_parameter = "/rtfm/bb_at"` in `rtfm.auto.tfvars`, then seed the real token ([Authentication](./authentication.md)).
3. **Build the five Lambda zips** into `terraform/app/.build/` ([Terraform Layers](./terraform-layers.md#lambda-build)).
4. **data** — `terraform apply` in `qa-data` (tables, EFS, network connector).
5. **app** — `terraform apply` in `qa` (front door, Lambdas, space seed items, DNS).
6. **Launch the space** (or wait for the 15-minute reconcile):

   ```bash
   aws lambda invoke --function-name rtfm-qa-lifecycle-manager \
     --payload '{"space":"platform"}' --cli-binary-format raw-in-base64-out \
     --profile rtfm-non-prod /tmp/out.json
   ```

7. **Create a Cognito user** (admin-create-only pool) — see [Authentication](./authentication.md).

Always save plan/apply output under `logs/<service>/` and stop on any Terraform failure.

### Promoting an existing image artifact (qa to prod)

Copy the proven zip cross-account and `create-microvm-image` against it, so prod runs identical bytes:

```bash
aws s3 cp s3://rtfm-non-prod-microvm-artifacts/rtfm-microvm.zip /tmp/ --profile rtfm-non-prod
aws s3 cp /tmp/rtfm-microvm.zip s3://rtfm-prod-microvm-artifacts/ --profile rtfm-prod
aws lambda-microvms create-microvm-image --name rtfm \
  --code-artifact uri=s3://rtfm-prod-microvm-artifacts/rtfm-microvm.zip \
  --base-image-arn arn:aws:lambda:us-east-1:aws:microvm-image:al2023-1 \
  --build-role-arn arn:aws:iam::288513693345:role/rtfm-prod-microvm-build \
  --cpu-configurations architecture=ARM_64 --resources minimumMemoryInMiB=4096 \
  --additional-os-capabilities '["ALL"]' \
  --hooks 'port=9000,microvmHooks={run=ENABLED,runTimeoutInSeconds=60},microvmImageHooks={ready=ENABLED,readyTimeoutInSeconds=300}' \
  --profile rtfm-prod
```

Prod's version is its own auto-assigned number (1.0), not non-prod's.

## DNS

RTFM is reached internally through the **private** hosted zone (`rtfm-non-prod.toluna.me`), whose `<env>` record is a CNAME to the shared execute-api VPC endpoint. For corporate resolvers to answer it, the zone **must be associated with the shared VPC** the resolver forwards into (non-prod: `vpc-04d454f7eb539d7c3`, owned by `platform-net`).

- `switch-traffic` only reads the zone and writes records into it. It does **not** associate the zone with a VPC; that is a prerequisite.
- Cross-account association: authorize from `rtfm-non-prod`, then `associate-vpc-with-hosted-zone` from `platform-net`.
- Symptom of a missing association: NXDOMAIN. Clients may hold a 24-hour negative cache; flush with `sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder` and `chrome://net-internals/#dns`.
- For prod, associate the prod private zone with the prod shared VPC (`platform-prod2`) **before** expecting resolution.

## Current deployment state

### qa (non-prod, 525997793977)

- **Image** `rtfm`, latest active version 6.0 (4 GB memory, all OS capabilities, EFS native-nfs4 entrypoint, git `safe.directory` fix).
- **shared** (`shared-non-prod`): SSM params, build bucket and role, Cognito pool and client with callbacks for qa/stg/dev/idan.
- **data** (`qa-data`): tables `rtfm-qa-data-{state,routes,ws-connections}`, EFS, mount targets, security group and VPC egress network connector.
- **app** (`qa`): private REST API `rtfm-qa-apigw`, WebSocket API, five Lambdas, EFS access point `/spaces/qa/platform`, seed item `qa#platform`, EventBridge schedules, private custom domain `qa.rtfm-non-prod.toluna.me`.
- **Space `platform`**: running, serves HTTP 200, EFS mounted at `/content`, content persists across restarts.

### prod (288513693345) — partially live

- Image `rtfm` v1.0, built by promoting the exact qa artifact.
- App layer: private REST API `rtfm-prod-apigw`, stage `prod`, five Lambdas.
- `https://prod.rtfm.toluna.me/platform/` and `https://rtfm.toluna.me/platform/` return HTTP 200, internally through ALB `tribe-net-gw-alb-prod2` to the execute-api VPC endpoint.
- Still to do: the full `shared-prod` / `prod-data` / `prod` Terraform run, the prod private-zone association check, and launching the prod `platform` space. See [Known Issues and Open Work](./known-issues-and-open-work.md).

## Troubleshooting: front door 403 from API Gateway

**Symptom:** `403 ForbiddenException` with no `x-amzn-remapped-*` / `x-amzn-trace-id` headers, although DNS, ALB targets, VPC endpoint policy and resource policy all look correct.

**Cause:** the VPCE access association (`aws_api_gateway_domain_name_access_association`) is bound to a **dead domain-name id** left over from a custom-domain recreate. A private API refuses admission unless the request's VPC endpoint is associated with the **live** domain.

**Fix (from the net account, `platform-net`):**

1. Add the live domain to the RAM share `rtfm_<env>-custom_domain`.
2. Delete the stale association.
3. Create the association `<live-domain> -> <vpce-id>`.
4. Wait 30–60 s for propagation.

> A manual fix is drift from Terraform. Run the app apply so `switch-traffic` recreates the domain, association and RAM share together; otherwise the next apply can re-break the front door.

## Cleanups already performed

- Removed orphan API Gateway custom domains and stale API mappings pointing at deleted APIs.
- Removed stale private-zone CNAMEs pointing at deleted endpoints.
- Removed stale VPCE access associations (including two prod ones) and recreated them against the live domains.
- Re-pointed the non-prod private zone from a phantom VPC to the shared VPC.

## Known perpetual plan diffs (cosmetic)

The Monty skeleton (`lambda_health`, `pipeline`, and `switch-traffic`'s two custom-domain resources) produces a non-empty plan on repeat applies (deployment redeploy trigger, policy JSON re-sorting, DNS record normalization). These are not RTFM resources and are not destructive.
