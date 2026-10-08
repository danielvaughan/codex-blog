---
title: "Amazon Bedrock for GPT-6: Routing Codex CLI Through AWS in Enterprise Environments"
parent: "Articles"
nav_order: 1170
date: 2026-09-26T08:00:00+00:00
last_modified_at: 2026-10-08T03:14:20+01:00
tags: ["codex-cli", "amazon-bedrock", "gpt-6", "sol", "luna", "enterprise", "aws", "data-residency", "model-routing", "v0.157.0"]
---

# Amazon Bedrock for GPT-6: Routing Codex CLI Through AWS in Enterprise Environments


---

Codex CLI v0.157.0, released 25 September 2026, shipped the first stable integration between Codex CLI and Amazon Bedrock for GPT-6 Sol and Luna.[^1] Enterprise organisations that could not adopt GPT-6 because their data policies require AWS infrastructure now have a supported migration path. Traffic routes through your existing AWS region; the model capability is identical to the OpenAI-hosted tier.

This article explains what Bedrock routing does, how to configure it, which models are supported, and how it interacts with the cost-governance and multi-agent configuration options introduced in recent Codex CLI releases.

## Background: Bedrock Integration in Codex CLI

Amazon Bedrock support in Codex CLI has a longer history than the v0.157.0 headline suggests. The foundation was laid in v0.148.0 (August 2026), when OpenAI and AWS announced general availability of GPT-5.6 and Codex on Amazon Bedrock.[^2][^5] That release introduced the Bedrock runtime provider, AWS credential refresh via configured commands, and Responses API compaction for Bedrock sessions — the same compaction behaviour used on the OpenAI-hosted tier.

v0.149.0 extended multi-agent support to Bedrock models, using the multi-agent V1 protocol for cross-provider orchestration. v0.153.3 added GPT-6 Astra to the Bedrock model picker as an experimental feature, making it accessible to enterprise teams without switching from AWS.

v0.157.0 completes this progression: GPT-6 Sol and Luna are now stable, first-class options on Bedrock. The full GPT-6 tier is available through AWS infrastructure.

## What Bedrock Routing Does and Does Not Change

Routing through Bedrock affects where your inference traffic terminates — it does not affect what the model does.

GPT-6 Sol on Bedrock produces the same outputs as GPT-6 Sol on the OpenAI API. The model weights, capability envelope, and context window are identical. The routing change means your prompts, completions, and file content travel via AWS infrastructure rather than OpenAI's directly. For organisations subject to AWS data-processing agreements, this is the critical distinction: data stays within the AWS boundary your legal and security teams have approved.

What routing through Bedrock does not change:

- Model capability and output quality
- Context window size
- The Codex CLI tool set — all hooks, approval policies, and AGENTS.md features work identically
- Compaction behaviour — Responses API compaction runs on Bedrock sessions as it does on OpenAI-hosted sessions
- Multi-agent V1 protocol support

What it does affect:

- Credential chain — Bedrock uses AWS credentials rather than an OpenAI API key
- Latency — routing via AWS may add single-digit milliseconds depending on region proximity
- Billing — usage appears in your AWS Cost Explorer rather than the OpenAI billing dashboard
- Regional data residency — traffic terminates in the AWS region you specify

## Supported Models at v0.157.0

At the v0.157.0 stable release, the following GPT-6 models are confirmed on Bedrock:

| Model | Bedrock status | Picker name |
|-------|---------------|-------------|
| GPT-6 Sol | Stable (v0.157.0) | `gpt-6-sol` |
| GPT-6 Luna | Stable (v0.157.0) | `gpt-6-luna` |
| GPT-6 Astra | Experimental (since v0.153.3) | `gpt-6-astra-*` |

Astra-tier models on Bedrock remain subject to availability constraints by region. Sol and Luna have broader regional coverage. Check the Bedrock model availability matrix in the AWS console for your specific region before configuring production sessions.

## Configuration

Bedrock routing is configured in `config.toml`. You specify the provider, AWS region, and model:

```toml
[model_routing]
provider = "bedrock"
region   = "us-east-1"

# Primary model for interactive sessions
model = "gpt-6-sol"

# Lightweight model for subagents
[agents]
default_subagent_model = "gpt-6-luna"
```

Credentials are resolved through the standard AWS credential chain: environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`), `~/.aws/credentials`, or IAM role assignment. Codex CLI v0.148.0+ handles credential refresh for expiring session tokens — the TUI surfaces a progress indicator during reauthentication rather than silently hanging.

If you are upgrading from older GPT-5.x models on Bedrock, Codex CLI v0.157.0 ships with in-CLI migration prompts that detect your current model configuration and propose updated model identifiers. Accept the suggestion or edit the config block manually — either path produces a valid `config.toml`.

### Combining Bedrock Routing with Rollout Budgets

The `[features.rollout_budget]` block works identically under Bedrock routing. Token accounting draws from the same shared ledger regardless of provider:

```toml
[model_routing]
provider = "bedrock"
region   = "eu-west-1"

model = "gpt-6-sol"

[agents]
default_subagent_model = "gpt-6-luna"

[features.rollout_budget]
enabled                     = true
limit_tokens                = 200000
reminder_at_remaining_tokens = [50000, 20000]
```

This configuration routes all traffic through AWS `eu-west-1`, uses Sol for the primary agent and Luna for subagents (cost differentiation by task complexity), and enforces a 200k-token session ceiling with reminders at 50k and 20k remaining. The cost governance approach from the Rollout Token Budgets article applies here without modification — Bedrock routing is transparent to the budget accounting layer.[^3]

## Enterprise Use Cases

The organisations most directly unblocked by GPT-6 on Bedrock fall into three categories.

**Financial services.** Tier-1 banks and asset managers operate under data-sovereignty frameworks (UK FCA, EU DORA, US OCC) that constrain where inference traffic may terminate. AWS infrastructure — particularly dedicated tenancy regions — satisfies these frameworks in a way that OpenAI-direct routing does not. Teams at these organisations were able to use GPT-5.6 on Bedrock since v0.148.0; v0.157.0 extends the same capability to GPT-6.

**Healthcare and life sciences.** HIPAA Business Associate Agreements are standard for US healthcare organisations. AWS offers BAAs covering Bedrock; OpenAI's API BAA coverage differs. Teams building Codex CLI agents that process clinical notes, lab data, or patient-adjacent information can now operate under their existing AWS BAA.

**Government and public sector.** FedRAMP authorisation shapes US federal procurement. AWS GovCloud regions carry higher-level FedRAMP authorisations. Organisations awaiting an OpenAI FedRAMP path can operate via Bedrock in GovCloud today.

For these teams, the shift is not primarily a performance decision — Sol on Bedrock and Sol on OpenAI produce indistinguishable results. The decision is about which infrastructure boundary satisfies the applicable compliance regime.

## Connecting to the Three-Tier Routing Strategy

The three-tier model strategy introduced in v0.156.1 — Astra for hard problems with long context, Sol for standard agentic coding, Luna for lightweight tasks and subagents — applies unchanged when routing via Bedrock.[^4]

The configuration already shown above puts this into practice: Sol handles interactive sessions (where mid-tier capability and speed are right), Luna handles subagent invocations (where throughput and cost matter more than capability), and Astra is available for sessions where cross-context memory notes are required (opt-in, higher credit rate).

The only Bedrock-specific consideration in the routing strategy is model regional availability. If your required AWS region does not yet carry GPT-6 Astra, fall back to Sol for long sessions rather than switching region — cross-region inference introduces latency and may not satisfy the data-residency requirement you are routing through Bedrock to meet.

## What Does Not Change for Your Workflow

For developers not constrained by data-residency requirements, Bedrock routing adds configuration without adding capability. There is no reason to prefer Bedrock over OpenAI-direct if your organisation's policies permit direct API access.

The tooling — hooks, approval policies, codex queue, AGENTS.md files, the `/usage` dashboard, rollout budgets — works identically in both configurations. An AGENTS.md file written for an OpenAI-hosted session works without modification when you switch provider to Bedrock. The only behavioural difference is in billing and in the network path your traffic takes.

## Verifying Your Setup

After configuring Bedrock routing, verify the connection before running production sessions:

```bash
# Confirm the active model and provider
codex config show

# Run a minimal test session to verify auth and routing
codex exec --model gpt-6-sol "Print the current date."

# Check token usage in the /usage dashboard after the test
# Open a session and run: /usage
```

If credential resolution fails, the CLI surfaces an auth error rather than silently falling back to OpenAI-direct. Check `AWS_PROFILE` and region configuration if the test session returns a `CredentialsNotFound` or `ModelNotAvailable` error for your target region.

---

**Companion articles:**
- [GPT-6 Sol and Luna in the Model Picker: Three-Tier Routing for Codex CLI](2026-09-25-gpt-6-sol-luna-model-picker-three-tier-routing-codex-cli.md)
- [Daemon Auto-Start Is Now the Stable Default in Codex CLI v0.157.0](2026-09-25-codex-cli-daemon-auto-start-stable-default-v0157.md)
- [Rollout Token Budgets and Multi-Agent Delegation: Cross-Thread Cost Governance in Codex CLI](2026-09-09-rollout-token-budget-multi-agent-delegation-cross-thread-cost-governance-codex-cli.md)

---

[^1]: OpenAI, "Codex CLI v0.157.0 Release Notes," GitHub, 25 September 2026. https://github.com/openai/codex/releases/tag/v0.157.0

[^2]: AWS, "OpenAI Models and Codex on Amazon Bedrock Are Now Generally Available," AWS Machine Learning Blog, August 2026. https://aws.amazon.com/blogs/machine-learning/openai-models-and-codex-on-amazon-bedrock-are-now-generally-available/

[^3]: Vaughan, D., "Rollout Token Budgets and Multi-Agent Delegation: Cross-Thread Cost Governance in Codex CLI," danielvaughan.com/codex-resources, 9 September 2026. https://danielvaughan.com/codex-resources/articles/2026-09-09-rollout-token-budget-multi-agent-delegation-cross-thread-cost-governance-codex-cli/

[^4]: Vaughan, D., "GPT-6 Sol and Luna in the Model Picker: Three-Tier Routing for Codex CLI," danielvaughan.com/codex-resources, 25 September 2026. https://danielvaughan.com/codex-resources/articles/2026-09-25-gpt-6-sol-luna-model-picker-three-tier-routing-codex-cli/

[^5]: OpenAI, "Codex CLI v0.148.0 Release Notes — Amazon Bedrock General Availability," GitHub, August 2026. https://github.com/openai/codex/releases/tag/v0.148.0
