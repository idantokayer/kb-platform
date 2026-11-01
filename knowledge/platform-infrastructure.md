---
title: VII - Platform Infrastructure12
description: Reference hub for the platform team's Terraform modules, internal tools, CI/CD pipelines, infrastructure layers, and microservice templates.
tags: [plm, platform, infrastructure, terraform, index]
---

# VII - Platform Infrastructure

## What this section is about12

The complete reference for all Terraform modules maintained by the platform team. This is where developers find documentation on how to use our modules in their services.

## Sections

- [Terraform Modules](./terraform-modules/index.md) — every Terraform module maintained by the platform team, with purpose, architecture, usage, inputs/outputs, dependencies, and release notes.
- [Internal Tools](./internal-tools/index.md) — standalone applications and services the platform team builds for the wider R&D organization.
- [Pipelines](./pipelines/index.md) — pipeline types and the end-to-end deployment flow.
- [Terraform Infrastructure layers](./terraform-infrastructure-layers.md) — how every microservice is split into shared, app, and data layers.
- [Platform microservice templates](./platform-microservice-templates/index.md) — reference implementations for building microservices on the platform.

## What each module child page represents

A single Terraform module with:

- Purpose and what problem it solves
- Architecture overview
- Usage examples (HCL snippets)
- Inputs/outputs reference
- Dependencies on other modules
- Limitations and known constraints

Each module page has a **"Release Notes"** child page tracking version history — what changed, when, and any migration steps required.

## How to use

When working with your service's Terraform code, look up the module here to understand its configuration options and behavior. Check the Release Notes page when upgrading to a new version to understand what changed and if any action is needed on your side.
