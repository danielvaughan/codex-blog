---
title: "AWS Agent Toolkit for AWS: Enterprise MCP, Skills, and Plugins for Codex CLI"
description: "On 6 May 2026 AWS launched the Agent Toolkit for AWS, consolidating its scattered agent infrastructure into a single official bundle of MCP servers, skills."
date: 2026-05-09T00:00:00+00:00
last_modified_at: 2026-06-12T21:40:50+01:00
categories: [ecosystem, enterprise, MCP]
tags: [aws, agent-toolkit, mcp, skills, plugins, enterprise, codex-cli]
parent: "Articles"
nav_order: 933
---

![Sketchnote diagram for: AWS Agent Toolkit for AWS: Enterprise MCP, Skills, and Plugins for Codex CLI](/sketchnotes/articles/2026-05-09-aws-agent-toolkit-codex-cli-enterprise-mcp-skills-plugins.png)


# AWS Agent Toolkit for AWS: Enterprise MCP, Skills, and Plugins for Codex CLI

On 6 May 2026 AWS launched the [Agent Toolkit for AWS](https://github.com/aws/agent-toolkit-for-aws), consolidating its scattered agent infrastructure into a single official bundle of MCP servers, skills, and plugins. For Codex CLI users building enterprise workflows, this is the most significant AWS-side development since the cli-agent-orchestrator.

## What shipped

The toolkit has three layers:

**AWS MCP Server (GA)** --- A managed Model Context Protocol server providing full AWS API access with enterprise governance baked in. Read-only by default, with approval gates for write operations. IAM condition keys distinguish agent-initiated actions from human ones, meaning CloudTrail logs can answer "did a human or an agent create that Lambda?" CloudWatch metrics track agent activity volumes.

**Skills** --- Curated workflow packages loaded on demand. Current skills cover serverless API deployment to Lambda, CloudFront + S3 site setup, CDK project scaffolding with proper IAM scoping, container orchestration, storage configuration, observability setup, and billing analysis. Skills are designed to be lean --- agents discover and retrieve only what is relevant to the current task, keeping context windows manageable.

**Plugins** --- Agent-specific integrations that bundle MCP config with skills. Three plugin bundles ship at launch: aws-core (foundational AWS access), aws-agents (agentic workflow patterns), and aws-data-analytics (data pipeline tooling).

## Codex CLI setup

Installation is straightforward via the plugin marketplace:

```bash
codex plugin marketplace add aws/agent-toolkit-for-aws
```

Then launch Codex and run `/plugins` to browse and install the aws-core plugin. Skills can also be installed standalone:

```bash
npx skills add aws/agent-toolkit-for-aws/skills
```

## Supported agents

The toolkit supports Claude Code, Codex, Kiro, Cursor, Cline, and Windsurf out of the box. Any MCP-compatible agent can use the AWS MCP Server directly.

## Enterprise governance story

This is where the toolkit earns its weight for Daniel's agentic pod work:

- **IAM condition keys** let security teams write policies that apply differently to agent and human actions --- for example, allowing agents to read but not delete production resources
- **CloudTrail audit logging** provides a complete trail of agent-initiated AWS API calls, essential for regulated environments
- **CloudWatch metrics** surface agent activity volumes for capacity planning and anomaly detection
- **Read-only default mode** with explicit approval gates prevents agents from making unintended infrastructure changes
- **Plugin-based distribution** means platform teams can enforce consistent AWS tooling across all agents via shared plugin sets (leveraging Codex v0.129+ plugin sharing and workspace controls)

## How it fits the broader stack

The toolkit sits at the infrastructure access layer. Combined with:

- **awslabs/cli-agent-orchestrator** (553 stars) for multi-agent coordination via tmux + MCP supervisor-worker pattern
- **Codex v0.130's `codex remote-control`** for headless agent services
- **Codex v0.130's Bedrock auth improvements** (#21623) for AWS console-login credentials
- **v0.131-alpha's Bedrock Mantle client header** (#21840) for agent identification

...AWS users now have a coherent stack from orchestration through to governed infrastructure access.

## Relationship to awslabs/mcp

The [awslabs/mcp](https://github.com/awslabs/mcp) repository continues to host individual AWS service MCP servers (Bedrock, CDK, ECS, Lambda, DynamoDB, RDS). These are maintained alongside the toolkit rather than deprecated. The toolkit provides the umbrella governance layer; individual MCP servers remain available for fine-grained service access.

## Key numbers

- 496 GitHub stars at time of capture (3 days post-launch)
- Apache-2.0 licensed
- Three plugin bundles at launch
- Supports 7+ AI coding agents

## Why it matters

For enterprise teams evaluating Codex CLI for production use, the AWS Agent Toolkit answers the "but how do we govern agent access to AWS?" question with an officially supported, auditable solution. Combined with the cli-agent-orchestrator for multi-agent coordination, AWS now offers a complete enterprise agent stack.

---

Sources: [github.com/aws/agent-toolkit-for-aws](https://github.com/aws/agent-toolkit-for-aws), [aws.amazon.com/agent-toolkit](https://aws.amazon.com/products/developer-tools/agent-toolkit-for-aws/), [andrew.ooo analysis](https://andrew.ooo/answers/what-is-aws-agent-toolkit-mcp-skills-plugins-may-2026/)
