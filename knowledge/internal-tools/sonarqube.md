---
title: SonarQube - Code Quality & Security Analysis
description: Toluna's self-hosted SonarQube — static code analysis and quality gates built into every CI/CD build, blocking deployments when code fails quality or security standards.
tags: [plm, internal-tools, sonarqube, code-quality, security, ci-cd]
---

# SonarQube - Code Quality & Security Analysis

Part of [Internal Tools](./index.md).

## What it does

**SonarQube** is Toluna's self-hosted static code analysis platform that performs continuous inspection of code quality and security vulnerabilities. It runs as part of every CI/CD pipeline build, acting as a quality gate that can block deployments when code does not meet defined standards.

The platform provides:

- **Code Quality Analysis** — Detects bugs, code smells, and maintainability issues across all supported languages
- **Security Vulnerability Detection** — Identifies security hotspots and known vulnerability patterns (OWASP, CWE)
- **Code Coverage Tracking** — Aggregates unit test coverage reports and enforces minimum thresholds
- **Quality Gates** — Configurable pass/fail criteria that block CI pipelines when code doesn't meet standards
- **Technical Debt Tracking** — Measures and trends technical debt over time per project

## What problem it solves

- **Inconsistent code quality**: SonarQube enforces consistent standards across all teams and repositories.
- **Security blind spots**: Automated SAST (Static Application Security Testing) finds patterns that humans miss.
- **Late defect discovery**: Running analysis at build time and blocking the pipeline on quality gate failure catches issues before they reach any environment.
- **Visibility**: A centralized dashboard showing code health across all R&D projects.

## Architecture overview

### Infrastructure

| Component | Technology | Purpose |
| --- | --- | --- |
| Application | SonarQube Community Edition (Docker) | Code analysis engine and web dashboard |
| Compute | AWS ECS (Fargate) | Container orchestration |
| Database | PostgreSQL 14 on RDS (db.t4g.medium) | Persistent storage for analysis results |
| Load Balancer | ALB (public-facing, IP-whitelisted) | HTTPS termination and blue/green routing |
| Deployment | Blue/Green via CodeDeploy | Zero-downtime deployments |
| Container Registry | ECR (cross-account replication) | Docker image storage |
| DNS | Route 53 (toluna-internal.com) | Internal domain resolution |
| CI/CD | AWS CodePipeline + CodeBuild | Automated build and deployment |
| IaC | Terraform (3 layers: shared, data, app) | Infrastructure management |
| Observability | Datadog + CloudWatch | Monitoring and logging |

### Network & Security

SonarQube's ALB is IP-whitelisted to restrict access to AWS CodeBuild NAT IPs (so CI pipelines can submit analysis reports), Azure DevOps IPs, and Corporate VPN CIDRs (for developer access). Whitelisted CIDRs are managed centrally via an SSM parameter and applied to the ALB security group.

### Deployment Model

- **Non-prod**: Development/QA instance for testing SonarQube configuration changes
- **Production**: The primary instance used by all pipelines — `prod.sonar.toluna-internal.com`

## How it integrates with platform microservices

SonarQube analysis is **built into the standard CI/CD pipeline** for every microservice. It runs during the Docker build phase in CodeBuild — no additional pipeline configuration is needed by service teams.

### Integration flow

1. Developer pushes code to Bitbucket
2. CodePipeline triggers CodeBuild
3. CodeBuild fetches `SONAR_HOST` and `SONAR_TOKEN` from SSM Parameter Store
4. During `docker build`, these are passed as build arguments (`--build-arg`)
5. Inside the Dockerfile, the sonar scanner runs analysis and submits results to SonarQube
6. SonarQube evaluates the quality gate and returns pass/fail
7. If the quality gate **fails**, the Docker build exits with code 1, **blocking the pipeline**

### SSM Parameters (pre-configured per account)

| Parameter | Purpose |
| --- | --- |
| `/infra/sonarqube/host` | SonarQube server hostname (e.g., `prod.sonar.toluna-internal.com`) |
| `/infra/sonarqube/token` | Authentication token for submitting analysis |

### Example: .NET service (CandyNet template)

In the **buildspec.yml.tpl**, SSM parameters are loaded:

```yaml
env:
  parameter-store:
    SONAR_HOST: "/infra/sonarqube/host"
    SONAR_TOKEN: "/infra/sonarqube/token"
```

Then passed to Docker build:

```shell
docker build -t ${IMAGE_URI} \
  --build-arg SONAR_HOST=$SONAR_HOST \
  --build-arg SONAR_TOKEN=$SONAR_TOKEN \
  --build-arg APP_VERSION=$(git rev-parse --short HEAD) .
```

Inside the **Dockerfile**, the scanner runs during the build stage:

```dockerfile
# Begin SonarQube analysis
RUN if [ -n "${SONAR_HOST}" ]; then \
    dotnet sonarscanner begin /k:"MyProject" \
      /d:sonar.host.url="https://$SONAR_HOST" \
      /d:sonar.qualitygate.wait=true \
      /d:sonar.token="$SONAR_TOKEN" \
      /v:$APP_VERSION \
      /d:sonar.cs.vstest.reportsPaths="*.trx" \
      /d:sonar.coverageReportPaths="./testresults/SonarQube.xml"; \
fi

# Run tests
RUN dotnet test -c Release --logger trx --collect:"XPlat Code Coverage"

# End SonarQube analysis (submits results, checks quality gate)
RUN dotnet sonarscanner end /d:sonar.token="$SONAR_TOKEN"
# If quality gate fails, exit 1 → build fails
```

### Example: Python service

```dockerfile
RUN curl https://${SONAR_HOST}/api/server/version; \
    if [[ $(echo $?) -eq 0 ]]; then \
        sonar-scanner -Dsonar.host.url=https://$SONAR_HOST \
        -Dsonar.projectKey=MyProject \
        -Dsonar.projectVersion=$APP_VERSION \
        -Dsonar.token=$SONAR_TOKEN \
        -Dsonar.qualitygate.wait=true; \
        if [[ $(echo $?) -ne 0 ]]; then \
            echo "SonarQube did not pass quality gate"; \
            exit 1; \
        fi \
    else echo "Sonar server not available skipping sonar scan"; \
    fi
```

### Key behaviors

- **Graceful degradation**: If SonarQube is unreachable (e.g., during maintenance), the build continues without analysis rather than failing.
- **Quality gate enforcement**: When SonarQube IS reachable, a failing quality gate **will block** the build (`exit 1`).
- **Runs in Docker build stage**: Analysis runs in a multi-stage Docker build, so results never persist in the final runtime image.
- **Project key convention**: Each service registers with a unique project key (typically the service/repo name).

## How to access it

**VPN Required**: To access the dashboard at [https://prod.sonar.toluna-internal.com/projects](https://prod.sonar.toluna-internal.com/projects), you must be connected to the corporate VPN. The ALB only allows traffic from whitelisted CIDRs.

On the dashboard you can view project analysis results and quality gate status, drill into specific issues (bugs, vulnerabilities, code smells), review code coverage reports, track quality trends, and (admin only) configure quality profiles and gates.

## Supported languages

C# / .NET (primary), Python, JavaScript / TypeScript, Java / Kotlin, Go, HTML, CSS, XML.

## Key design decisions

- **Self-hosted (not SaaS)** — Runs inside Toluna's AWS infrastructure to keep source code and analysis results within the network boundary.
- **Build-time integration (not PR decoration)** — Analysis runs inside the Docker build phase, ensuring every built image has passed quality gates.
- **IP whitelisting** — ALB restricted to CodeBuild IPs + Azure DevOps IPs + VPN CIDRs.
- **Blue/Green deployment** — Zero-downtime deployments for SonarQube itself.
- **Graceful skip when unreachable** — Builds check availability first (`curl .../api/server/version`); if down, the build proceeds without analysis.
- **Centralized SSM parameters** — Host and token stored in SSM per account.

## Technology stack

| Layer | Technology |
| --- | --- |
| Application | SonarQube Community Edition (Java) |
| Database | PostgreSQL 14 (AWS RDS) |
| Compute | AWS ECS Fargate (512 CPU / 4096 MB memory) |
| Networking | ALB (HTTPS/443) + blue/green target groups |
| Container | ECR with cross-account replication (non-prod → prod) |
| Infrastructure | Terraform (shared/data/app layers) |
| CI/CD | AWS CodePipeline + CodeBuild + CodeDeploy |
| DNS | Route 53 (toluna-internal.com zone) |
| Secrets | AWS SSM Parameter Store |
| Forwarder | Lambda (Python, urllib3) — HTTP request proxy |

## Repository

Source code: Bitbucket — `tolunaengineering/sonarqube`

Structure: `service/` (Dockerfile, docker-compose, config), `forwarder-lambda/` (HTTP forwarder Lambda), `terraform/app/` (ECS, ALB, pipelines), `terraform/data/` (RDS PostgreSQL), `terraform/shared/` (ECR, DNS, Datadog, pipeline controller, test framework), `tests/` (Postman + stress tests).

## Environments

| Environment | URL | Purpose |
| --- | --- | --- |
| Production | `prod.sonar.toluna-internal.com` | Primary instance for all CI pipelines |
| Non-prod | `{env}.sonar.toluna-internal.com` | Testing SonarQube upgrades and config changes |

## Who maintains it

Owned and maintained by the **Platform team**.
