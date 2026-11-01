---
title: Env1 - Platform Services Hub
description: The platform team's internal hub — a private Terraform registry, a web-based service config editor, and an MCP server exposing config and registry data to AI assistants.
tags: [plm, internal-tools, env1, terraform-registry, mcp, config]
---

# Env1 - Platform Services Hub

Part of [Internal Tools](./index.md).

## What it does

**Env1** is the platform team's internal services hub that provides three core capabilities to the R&D organization:

1. **Private Terraform Registry** — A fully compliant implementation of the [Terraform Registry Protocol](https://developer.hashicorp.com/terraform/internals/module-registry-protocol) that enables teams to host, version, and distribute private Terraform modules stored in Bitbucket repositories.
2. **Service Configuration Editor** — A web-based UI for viewing and editing service configuration JSON files stored in S3 buckets, organized by tribe/environment. Supports versioning, diff viewing, and role-based access control.
3. **MCP Server (AI Integration)** — Exposes service configuration read/write operations and Terraform module registry data to AI assistants (e.g., Kiro) via the Model Context Protocol.

## What problem it solves

- **Module distribution**: Before env1, sharing Terraform modules across teams required manual Git URL management and lacked version discovery. Env1 provides a standard `terraform init` experience with automatic version resolution.
- **Configuration management**: Service configurations (JSON) previously required direct S3 console access or CLI operations. The Config Editor provides a governed, auditable UI with tribe-based access control.
- **AI-assisted operations**: The MCP server allows AI agents to query module versions, read/write service configs, and explore the module registry without manual human intervention.

## Architecture overview

### Infrastructure

| Component | Technology | Purpose |
| --- | --- | --- |
| Compute | AWS Lambda (Python 3.13) | Serverless request handling |
| API Layer | AWS API Gateway (Private) | REST API with VPC-only access |
| Authentication | AWS Cognito + Azure AD | SSO with tribe/role mapping |
| Storage | S3 (versioned buckets) | Config files per tribe |
| Secrets | AWS SSM Parameter Store | Bitbucket OAuth credentials |
| IaC | Terraform (3 layers: shared, data, app) | Infrastructure management |
| Observability | Datadog + CloudWatch | Tracing, metrics, logs |
| CI/CD | AWS CodePipeline + CodeBuild | Automated deployments |

### Lambda Functions

| Function | Purpose |
| --- | --- |
| TF Registry | Module version discovery, download URLs, documentation |
| Health | Service health checks and dependency monitoring |
| S3 Get File | Read config files from tribe-specific S3 buckets |
| S3 Put File | Write/update config files with content-type detection |
| S3 Delete File | Remove config files (with access control) |
| S3 List Files | List config files filtered by tribe membership |
| S3 Get Versions | Retrieve historical versions of config files |
| Cognito Pre-Auth Trigger | Maps Azure AD groups to Cognito tribes/roles on login |
| Org Tree Param Updater | Periodically syncs AWS Organization structure for tribe mapping |

### API Endpoints

**Terraform Registry (modules.v1):**

- `GET /.well-known/terraform.json` — Service discovery
- `GET /modules.v1/{namespace}/modules` — List available modules
- `GET /modules.v1/{namespace}/{module}/{provider}/versions` — Get module versions
- `GET /modules.v1/{namespace}/{module}/{provider}/{version}/download` — Get download URL
- `GET /modules.v1/{namespace}/{module}/{provider}/{version}/info` — Module documentation

**Config Editor (s3_storage):**

- `GET /s3_storage/{path}` — Read a config file
- `PUT /s3_storage/{path}` — Write/update a config file
- `DELETE /s3_storage/{path}` — Delete a config file
- `GET /s3_storage` — List files across accessible buckets
- `GET /s3_storage/{path}/versions` — Get file version history

**UI & Health:**

- `GET /ui` — Service Config Editor web application
- `GET /health_api` — Health check endpoint

### Authentication & Authorization

Env1 uses a layered authentication model:

1. **Azure AD SSO** — Users authenticate via corporate Azure AD through Cognito Hosted UI.
2. **Cognito Pre-Auth Trigger** — On login, a Lambda queries the AWS Organization tree and maps the user's Azure AD groups to `custom:tribes` and `custom:role` claims in the JWT.
3. **Role-Based Access**:
   - **Admin** (DevOpsSec group members) — Full access to all tribes and configurations
   - **User** — Access restricted to S3 buckets matching their tribe membership
4. **Tribe-Based Bucket Isolation** — Each tribe's configs live in a dedicated S3 bucket (`toluna-{tribe}-tf-config-data`). Users can only access buckets for tribes they belong to.

### Data Flow — Terraform Registry

1. Terraform client runs `terraform init` pointing to `env1.toluna.me`
2. API Gateway routes the request to the TF Registry Lambda
3. Lambda authenticates with Bitbucket via OAuth 2.0 client credentials
4. Lambda queries Bitbucket API for repository tags (versions) or README (docs)
5. For downloads: returns an `X-Terraform-Get` header with a Git URL containing an embedded temporary access token

### Data Flow — Config Editor

1. User opens `/ui` in browser (VPN required — private API Gateway)
2. Cognito handles SSO login, returning a JWT with tribe claims
3. UI calls S3 storage Lambda endpoints to list/read/write configs
4. Lambda validates tribe membership against the requested path's environment segment
5. Authorized operations execute against the tribe-specific S3 bucket

## How to use it

### As a Terraform user

Add the private registry as a module source in your Terraform code:

```hcl
module "example" {
  source  = "env1.toluna.me/Env1/terraform-toluna-example-module/aws"
  version = "1.2.3"
}
```

The registry supports standard Terraform version constraints (`~>`, `>=`, etc.).

### As a Config Editor user

1. Connect to VPN (private API Gateway requires VPC access)
2. Navigate to `https://env1.toluna.me/ui`
3. Log in with your corporate Azure AD credentials
4. You will see configurations for the tribes your Azure AD groups belong to
5. Edit JSON configs directly in the browser with syntax highlighting and validation
6. View version history and diff previous versions

### As an AI assistant (MCP)

The env1 MCP server exposes the following tools to AI agents:

- `read_s3_json` — Read service configuration by key (e.g., `myservice/app`)
- `write_s3_json` — Write service configuration by key
- `get_module_versions` — List all versions of a Terraform module
- `get_module_info` — Get detailed module documentation
- `list_all_registry_modules` — List available modules in a namespace

### Bucket naming convention

Config buckets follow the pattern: `toluna-{tribe}-tf-config-data`. Bucket configurations are managed via SSM parameter: `/infra/env1-{env_type}/tf_config_buckets`

## Key design decisions

- **Private API Gateway** — Accessible only from within the VPC (requires VPN). No public internet exposure.
- **Lambda warm-up** — EventBridge triggers keep Lambda instances warm every 5 minutes to eliminate cold-start latency for critical Terraform operations.
- **Per-request token refresh** — Bitbucket OAuth tokens are cached across Lambda invocations but refreshed transparently when expired.
- **Tribe isolation** — Each tribe's data lives in its own S3 bucket. Access control is enforced at the Lambda layer based on JWT claims, not bucket policies.
- **SSM-driven bucket discovery** — Bucket mappings stored in SSM allow adding new tribes without code changes.

## Technology stack

| Layer | Technology |
| --- | --- |
| Backend | Python 3.13, aiohttp, asyncio, boto3, PyJWT |
| Frontend (new UI) | React 19, TypeScript, Vite, MUI, CodeMirror, ReactFlow, AWS Amplify |
| Frontend (legacy) | Static HTML |
| Infrastructure | Terraform, AWS Lambda, API Gateway, S3, Cognito, SSM, CodePipeline |
| Observability | Datadog (ddtrace, datadog_lambda), CloudWatch |
| Auth | Azure AD → Cognito User Pool (OAuth 2.0 / OIDC). User must be member of both "Domain Users" and "Engineering" groups in Azure AD |

## Repository

Source code: Bitbucket — `env1` repository in the platform project.

Structure:

- `service/` — Python Lambda handlers + React UI
- `terraform/app/` — Application infrastructure (Lambdas, API Gateway, pipelines)
- `terraform/data/` — Data layer (Cognito, S3 buckets, CodeBuild)
- `terraform/shared/` — Shared resources (Datadog, FinOps, DynamoDB dumps)
- `tests/postman/` — Integration test collections per environment

## Who maintains it

Owned and maintained by the **Platform / DevOpsSec** team.
