---
title: Driftwatch Test Framework
description: A behavioral regression testing framework for agentic systems — define scenarios as JSON, invoke the agent, inspect tool usage, and use an LLM judge to catch behavioral drift in CI.
tags: [plm, mcp, testing, driftwatch, agents, ci]
---

# Driftwatch Test Framework

**Owner:** Platform team · **Status:** Active (v0.1) · **Repo:** [tolunaengineering/driftwatch](https://bitbucket.org/tolunaengineering/driftwatch)

Part of [MCP Development](../index.md).

## What Is This?

**driftwatch** is a behavioral regression testing framework for agentic systems — the integration-test layer for AI agents.

As teams build agents from shared building blocks — MCP tool servers, behavioral skills, and synced knowledge bases — a small change to any one piece can silently alter how an agent behaves. driftwatch makes that drift visible. You define scenarios as JSON (an input plus behavioral assertions), and driftwatch invokes your agent, inspects its tool usage, and uses an LLM judge to evaluate whether it followed the expected methodology.

It is **runtime-agnostic** (Kiro, Codex, or any custom CLI agent via pluggable runners) and runs both interactively and as a **CI gate** that fails the build on regressions.

## Why It Matters

Every team in the MCP ecosystem composes agents from the same tools and skills, but there's no easy way to prove a change didn't break behavior. driftwatch lets you assert that your agent still searches the knowledge base before external tools, uses the correct environment tags, and refuses dangerous / out-of-scope operations — and enforce it in CI. Assertions are on **methodology** (what the agent does), not exact wording — so they catch real drift without being brittle.

## Installation

Requires Python 3.9+.

```shell
# Point pip at the platform-pypi CodeArtifact repo (one-time)
aws codeartifact login --tool pip --domain toluna-artifact-repository \
  --repository platform-pypi --profile toluna-main

pip install driftwatch            # driftwatch[bedrock] for the CI (Bedrock) judge
```

If the `driftwatch` command isn't found after install, its script dir isn't on your PATH. `python3 -m driftwatch.cli` always works without PATH setup.

## Quick Start

```shell
driftwatch init        # scaffolds driftwatch.toml + an example scenario
#   edit driftwatch.toml -> point it at your agent(s)
#   add scenarios under scenarios/

driftwatch             # runs the default profile - no flags needed
```

## Profiles

A **profile** is a named bundle of run settings. Run one with `driftwatch <profile>`; bare `driftwatch` runs the `default_profile`.

```toml
default_profile = "full"

[profiles.full]
parallel = "auto"

[profiles.quick]
category = "instance-behavior"

[profiles.ci]
judge = { name = "bedrock", model = "anthropic.claude-3-5-sonnet-20241022-v2:0" }
```

```shell
driftwatch            # full
driftwatch quick      # fast subset
driftwatch ci         # Bedrock judge, no kiro dependency
```

Precedence: CLI flag → profile value → config default.

## Key Concepts

| Concept | Summary |
| --- | --- |
| **Runner** | Pluggable adapter that invokes an agent and parses its tool calls. Built-in: `kiro` (default), `custom-command` (Strands / Codex / any CLI). |
| **Judge** | Pluggable LLM evaluator for behavioral criteria. Built-in: `kiro`, `bedrock` (no Kiro dependency — ideal for CI). Independent of the runner. |
| **Scenario** | JSON: an input + assertions (`tool_usage`, `output_contains`, `exit_code`, `llm_assertions`). Single- or multi-instance. |
| **Two-layer mocking** | Layer A: MCP injectable mock passthrough (`MOCK_MODE` env) for locally-run servers. Layer B: framework-owned tool-response mocks for remote servers. |
| **Parallelism** | `parallel = "auto"` — machine-aware, conservative. Override per-profile or with `--parallel`. |

## Commands

| Command | Does |
| --- | --- |
| `driftwatch` / `driftwatch <profile>` | Run default / named profile |
| `driftwatch list` | Scenarios + persisted status |
| `driftwatch explain <id>` | Scenario input + resolved assertions |
| `driftwatch info` | Resolved config, profiles, auto-parallelism |
| `driftwatch coverage` | Tag (skill/protocol) + tool coverage |
| `driftwatch reset` | Clear run logs |
| `driftwatch init` | Scaffold starter config + example scenario |

Run artifacts are written to `.driftwatch/logs/` (gitignored).

## CI Usage

```yaml
- run: pip install "driftwatch[bedrock]"
- run: driftwatch ci        # exit code 1 on any failure
```

## Status & Roadmap

- **v0.1** — core engine, kiro + custom-command runners, kiro + bedrock judges, profiles, auto-parallelism, `init` scaffolding, two-layer mocking. Proven as a full replacement for the Scout Investigation Platform test runner (14 scenarios → 36 tests).
- **Next** — Codex runner adapter, framework-owned (Layer B) mock fixtures for remote MCP servers, richer reporting (JSON/HTML).

## Contact

- **Platform team** — framework and onboarding
- **Repo / README** — [tolunaengineering/driftwatch](https://bitbucket.org/tolunaengineering/driftwatch)
