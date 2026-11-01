---
title: http-relay
description: Deploys an Envoy-based HTTP relay as an ECS Fargate service behind an internal NLB, used by high-traffic gateways to proxy HTTP traffic with full Datadog observability.
tags: [plm, terraform, module, http-relay, envoy, ecs, data-layer, gateway]
---

# http-relay

Deploys an Envoy-based HTTP relay as an ECS Fargate service behind an internal Network Load Balancer (NLB). Used by high-traffic gateways (AI Gateway, Spark Gateway) to proxy HTTP traffic through Envoy within a VPC, with full Datadog observability.

Part of [Terraform Modules](./index.md). Lives in the data layer — see [Terraform Infrastructure layers](../terraform-infrastructure-layers.md). Architecture reference: [High-Traffic Tribe Gateway](https://toluna.atlassian.net/wiki/spaces/SOA/pages/2748841985) (external Confluence).

---

## Quick Reference

| Property | Value |
| --- | --- |
| **Registry** | `env1.toluna.me/toluna/terraform-aws-http-relay/aws` |
| **Current Version** | 0.0.3 |
| **Repository** | [tolunaengineering/terraform-aws-http-relay](https://bitbucket.org/tolunaengineering/terraform-aws-http-relay) |
| **Present In** | Data layer (persistent, not destroyed during deployments) |

---

## Purpose

Provides a stateless Envoy proxy layer that sits between API Gateway and backend services. Used when you need:

- **Dynamic forward proxying** — Envoy resolves the `Host` header at request time, routing to any backend without pre-configured routes
- **Distributed tracing** — two-listener architecture ensures Datadog spans get the correct resource name from the request path
- **Long-running requests** — no timeouts (stream_idle_timeout: 0), supporting operations up to 15 minutes
- **HTTP/1.1 enforcement** — prevents protocol errors with API Gateway/ALB backends that don't support HTTP/2

The relay lives in the **data layer** because it's persistent infrastructure that must survive app-layer blue/green deployments.

---

## Architecture

```
API Gateway REST API (public, REGIONAL)
    │
    ▼ (VPC Link V2)
Private ALB (app layer — terraform-aws-alb-traffic-transform)
    │  Rewrites Host header + path based on endpoint_mapping/proxy_mapping
    ▼ (:443 TCP passthrough)
Internal NLB (data layer — this module)
    │
    ▼ (:8080 HTTP)
ECS Fargate Tasks (data layer — this module)
    ├── Envoy Container (port 8080)
    │   ├── front_listener :8080 — Lua injects x-envoy-decorator-operation
    │   └── tracing_listener :8081 — Datadog tracer + dynamic forward proxy
    └── Datadog Agent Sidecar (port 8126)
    │
    ▼ (HTTPS :443, dynamic Host-based routing)
Backend services (private toluna.me DNS)
```

### Why two listeners?

1. **front_listener (:8080)** — receives traffic from ALB. A Lua filter reads the request path and injects it as `x-envoy-decorator-operation`. This header must exist _before_ the Datadog tracer starts the span.
2. **tracing_listener (127.0.0.1:8081)** — receives the request with the decorator header set. The Datadog tracer uses it as the span's resource name. Handles tracing, access logging, and dynamic forward proxying.

---

## How Routing Works

The http-relay module itself does **no routing** — it's a dumb forward proxy that routes based on the `Host` header it receives.

Routing happens upstream in the **app layer** via `terraform-aws-alb-traffic-transform`, which:

1. Matches incoming requests against `endpoint_mapping` (regex path conditions) and `proxy_mapping` (prefix-to-host mapping)
2. Rewrites the `Host` header to the target backend (e.g., `gitz.toluna.me`)
3. Optionally rewrites the path (regex replacement)
4. Forwards to the NLB (this module)

Envoy then simply resolves the `Host` header via DNS and proxies to the backend over HTTPS.

The routing configuration lives in `app-env.json` (managed via env1) — see [High-Traffic Tribe Gateway](https://toluna.atlassian.net/wiki/spaces/SOA/pages/2748841985) for the full routing config format.

---

## Runtime Behavior

### At terraform apply time

- Creates CloudWatch log group with 7-day retention
- Creates NLB target group (port 8080/TCP, IP targets, health check on `/health`)
- Creates internal NLB with TCP:443 listener (no TLS termination — raw passthrough)
- Deploys ECS Fargate service (2 tasks) using pre-built Envoy image
- Configures Datadog sidecar with APM enabled
- Exports NLB private IPs (used by VPC Link targets in app layer)

### Observability

- **Tracing:** `envoy.tracers.datadog` native tracer → DD Agent sidecar → Datadog APM
- **Logs:** JSON access logs with `dd.trace_id`, `dd.span_id` for log-trace correlation
- **Health checks excluded:** `/health` filtered from both traces and logs
- **Log status:** Lua filter sets `error` (5xx/response flags), `warn` (4xx), `info` (normal)

---

## Module Interactions

### Inputs From

| Source | What It Provides |
| --- | --- |
| [config-manager](./config-manager.md) | `config.globals` (app_name, env_name, env_type, tribe), `config.net` (vpc_id, private_subnets) |
| Shared layer | Security group ID (e.g., `{app_name}-shared-sg-{env}`) |
| Secrets Manager | Datadog API key (`datadog_api_key` secret) |

### Outputs Consumed By

| Consumer | What It Uses |
| --- | --- |
| `terraform-aws-alb-traffic-transform` (app layer) | Discovers NLB by name pattern `nlb-gw-{app_name}-{data_env}` + tags |
| App layer (VPC Link) | `nlb_ips` — static private IPs for API Gateway VPC Link targets |

---

## Inputs

| Variable | Type | Required | Description |
| --- | --- | --- | --- |
| `config` | object | Yes | Config-manager output (must include `globals` and `net`) |
| `security_group` | string | Yes | Security group ID for NLB and ECS tasks |
| `datadog_api_key` | string (sensitive) | Yes | Datadog API key for the sidecar agent |

---

## Outputs

| Output | Description |
| --- | --- |
| `nlb_ips` | List of NLB private IPs (one per subnet). Used by VPC Link targets in app layer. |

---

## Deployed Services

| Service | Account | Tribe | Workspace |
| --- | --- | --- | --- |
| ai-gw | 811710375491 | spark (shared) | ai-gw-dev-data |
| spark-gw | 443938285619 | spark | spark-gw-dev-data |

Both use the same Envoy image: `014931512072.dkr.ecr.us-east-1.amazonaws.com/envoy:linux64`

---

## Usage Example

```hcl
# In the DATA layer (terraform/data/)
module "http_relay" {
  source  = "env1.toluna.me/toluna/terraform-aws-http-relay/aws"
  version = "0.0.3"

  config          = module.config
  security_group  = data.aws_security_group.shared-sg.id
  datadog_api_key = data.aws_secretsmanager_secret_version.datadog_api_key.secret_string
}
```

The NLB IPs output is then consumed by the app layer to configure VPC Link targets for API Gateway.

---

## Why Data Layer?

The http-relay runs in the **data layer** (not app layer) because:

- The Envoy proxy is **persistent infrastructure** — it should not be destroyed during blue/green app deployments
- The NLB's private IPs must be **stable** — VPC Links reference them by IP, and IP changes would break API Gateway routing
- The ECS service is **not managed by gitflow** (`is_managed_by_gitflow = false`) — it doesn't participate in the standard CI/CD pipeline
- Updates to the Envoy image/config are done manually via `terraform apply` on the data layer
