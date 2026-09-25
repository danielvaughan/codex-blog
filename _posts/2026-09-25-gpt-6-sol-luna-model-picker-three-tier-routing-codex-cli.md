---
title: "GPT-6 Sol and Luna in the Model Picker: Three-Tier Routing for Codex CLI"
parent: "Articles"
nav_order: 1166
date: 2026-09-25T08:00:00+00:00
last_modified_at: 2026-09-25T11:40:05+01:00
tags: ["codex-cli", "model-selection", "gpt-6", "astra", "sol", "luna", "model-routing", "cost-governance", "enterprise"]
---

# GPT-6 Sol and Luna in the Model Picker: Three-Tier Routing for Codex CLI


---

When Codex CLI v0.156.1 shipped on 23 September 2026, it added GPT-6 Sol and Luna as selectable options in the CLI model picker for the first time.[^1] That one change made the previous model-selection guidance obsolete. Sol was no longer a mid-range option nested between Terra and a distant Astra tier — it became the baseline for standard agentic work. Luna became the default rate-limit fallback, replacing the older step-down behaviour. And Astra established itself as the high-capability tier that organisations opt into deliberately.

Two days later, v0.157.0 added Amazon Bedrock routing for GPT-6 Sol and Luna, extending the three-tier model strategy from a feature of the OpenAI platform into a cross-cloud architecture decision.[^2] This article explains what the new picker does, how to configure each tier, and how to build a routing strategy that keeps cost and capability in alignment.

## What Changed in v0.156.1

Before v0.156.1, the CLI model picker offered GPT-5.6 tiers — Sol, Terra, and Luna — and GPT-6 Astra as a separate experimental feature that required opt-in in Enterprise environments.[^3] The model-selection surface was essentially two separate systems: a stable production tier and a gated preview tier.

v0.156.1 collapsed that distinction. GPT-6 Sol and Luna now appear directly in the picker alongside Astra. The rate-limit switch prompt — shown when the session approaches a usage ceiling — now recommends Luna by default instead of stepping down to an older model. This is a behavioural change, not just a UI change: sessions that previously degraded silently to GPT-5.x behaviour now stay within the GPT-6 family.

The practical effect for teams is that the default model stack is now entirely GPT-6. Terra is gone from the default tier list. The operative tiers are:

| Tier | Model | Use case | Relative cost |
|------|-------|----------|---------------|
| Astra | GPT-6 Astra (Light / Medium / Extra High) | Hard problems, long sessions, cross-context memory | Highest (Astra Light: 2× standard credits) |
| Sol | GPT-6 Sol | Standard agentic tasks, most coding work | Mid |
| Luna | GPT-6 Luna | Lightweight edits, subagent roles, rate-limit fallback | Lowest |

GPT-6 Astra carries Codex's cross-context memory feature — notes that survive context rolls and remain searchable within a task. Sol and Luna do not include this feature. If you need persistent within-task recall across long sessions, Astra is the only viable tier.

## Configuring the Model Picker

The `config.toml` model field accepts the full model identifier or the shorthand used by the picker:

```toml
# config.toml — set Sol as the session default
model = "gpt-6-sol"

# Or use Luna for lightweight tasks
model = "gpt-6-luna"

# Or target a specific Astra tier
model = "gpt-6-astra-medium"
```

For multi-agent sessions, the subagent model defaults independently from the session model. If you are running a large agent team and want subagents on Luna while keeping the orchestrator on Sol, the split is explicit:

```toml
[agents]
default_subagent_model = "gpt-6-luna"
```

This pattern matters for cost. In a typical parallel-agent session with four subagents executing file edits, all four subagents process substantially more token volume than the orchestrator does in steering them. Routing subagents to Luna while keeping the orchestrator on Sol reduces per-session cost without degrading the decision quality of the session leader.

## The Rate-Limit Fallback Change

The most operationally significant change in v0.156.1 is the new fallback default. Previously, when a session hit a usage limit, the picker stepped down to whichever older model was available — often GPT-5.4, which reached end-of-life on 31 August 2026. The result was sessions that silently degraded in capability while continuing to run.

The v0.156.1 behaviour is different: when the session model hits a rate limit, the picker now recommends GPT-6 Luna, not a step-down to an older generation. Luna's cost structure means that staying in the GPT-6 family at the Luna tier is cheaper per token than GPT-5.6 Sol was, so the rate-limit prompt has both a cost and a capability argument.

For teams that were relying on the old step-down behaviour as an implicit cost-control mechanism — letting sessions drift towards cheaper legacy models as they approached limits — this change breaks that assumption. The new fallback is Luna, which is a reasonable default, but any team that wants to control fallback behaviour explicitly should configure it:

```toml
[model_fallback]
rate_limit_model = "gpt-6-luna"
```

Alternatively, use `rollout_budget` to cap session token spend before the rate-limit threshold is reached:

```toml
[features.rollout_budget]
enabled = true
limit_tokens = 200000
reminder_at_remaining_tokens = [50000, 20000]
```

## Amazon Bedrock Routing — v0.157.0

v0.157.0, stable on 25 September 2026, added Amazon Bedrock routing for GPT-6 Sol and Luna.[^2] This is the first time Bedrock integration has reached a stable Codex CLI release, and it changes the architecture decision for enterprise teams significantly.

Bedrock routing means GPT-6 Sol and Luna calls can be processed within AWS infrastructure rather than via OpenAI's direct API. For organisations that have AWS as their approved cloud provider and require data residency or traffic inspection compliance, this removes the primary blocker to GPT-6 adoption in Codex CLI.

The migration prompt that ships with v0.157.0 guides teams upgrading from older models to the GPT-6 Bedrock-routed equivalents. The configuration uses a `bedrock` routing block:

```toml
[model_routing]
provider = "bedrock"
region = "us-east-1"

# Model selection within Bedrock routing
model = "gpt-6-sol"
```

The Bedrock integration applies to Sol and Luna at launch — Astra via Bedrock is not confirmed in the v0.157.0 release notes. Teams that need Astra-tier capabilities under AWS routing should monitor the v0.158.0 cycle.

## Building a Three-Tier Routing Strategy

With three selectable tiers and Bedrock routing available, the model-selection decision is now a proper routing architecture question, not a one-time configuration choice. The practical framework has three components: task classification, tier assignment, and fallback policy.

### Task Classification

The decision between Astra, Sol, and Luna should be driven by task characteristics, not by cost alone:

- **Astra** is appropriate when a task requires cross-context recall across a session longer than a single context window, when the problem is open-ended enough that the agent will need to retrieve earlier reasoning, or when the task type is sufficiently complex that a lower-capability model would require multiple retries. Astra's extra cost is a poor trade for short, bounded tasks.
- **Sol** is appropriate for the majority of coding work: feature implementation, refactoring, debugging, test writing. The capability difference between Sol and Astra is real but not decisive for well-scoped tasks with a clear success criterion.
- **Luna** is appropriate for subagent execution roles (running a command, reading a file, applying a diff), lightweight edits, code formatting, and documentation passes. Luna is also the right model for any agent that runs at high frequency — Guardian review, automated test validation, CI hooks — where per-call cost accumulates.

### Tier Assignment in AGENTS.md

For teams running multi-agent workflows, encoding tier assignments in AGENTS.md removes the per-session cognitive overhead:[^5]

```markdown
## Model Routing Policy

- **Orchestrator**: gpt-6-sol (session default)
- **Subagents (implementation)**: gpt-6-luna (set via agents.default_subagent_model)
- **Long-running analysis tasks**: gpt-6-astra-light (opt-in per task)
- **Guardian review**: gpt-6-luna (automatic, controlled by Guardian config)
- **Rate-limit fallback**: gpt-6-luna (configured in model_fallback block)
```

### Fallback Policy

The three-tier model means there is now a sensible degradation chain: if an Astra session hits a limit, fall back to Sol (not Luna) to preserve capability for complex tasks. If a Sol session hits a limit, fall back to Luna. Hard stops — using `rollout_budget` — are preferable to silent fallback for overnight runs where the operator is not present to observe degradation.

## Enterprise Deployment Considerations

For Enterprise accounts, Astra remains off by default — workspace administrators must enable it per workspace.[^3] This is unchanged from its introduction with GPT-6. Sol and Luna are available to all paid tiers (Plus, Pro, Business, Enterprise); Free and Go users continue on Terra.

The Bedrock routing option in v0.157.0 is likely to be the deciding factor for financial services and healthcare organisations that have maintained GPT-5.x deployments because of data-handling requirements. The migration path is: confirm Bedrock region availability, test Sol and Luna via Bedrock routing in a non-production workspace, validate session behaviour against existing AGENTS.md configuration, then roll out to production workspaces with the `model_routing.provider = "bedrock"` config block.

## Implications for Book Chapter 11

The arrival of GPT-6 Sol and Luna in the production picker, combined with Bedrock routing in v0.157.0, requires a complete rewrite of the model selection chapter. The chapter's current structure — built around GPT-5.6 tiers and Astra as a separate enterprise-only feature — no longer reflects production behaviour. The revised chapter should cover the three-tier routing architecture, the subagent-model split, the rate-limit fallback change, and the Bedrock routing configuration as a first-class deployment option.

---

[^1]: Codex CLI v0.156.1 release notes (23 September 2026). GPT-6 Sol and Luna added to the CLI model picker; rate-limit switch prompt now recommends Luna as default.
[^2]: Codex CLI v0.157.0 release notes (25 September 2026). Amazon Bedrock routing for GPT-6 Sol and Luna; migration prompts for teams upgrading from older models.
[^3]: Codex CLI changelog-watch.md (September 2026). GPT-6 Astra Enterprise opt-in policy; six-tier model structure (Terra Light, Sol Light, Sol Medium, Astra Light, Astra Medium, Astra Extra High); Astra cross-context memory feature.
[^5]: Codex CLI AGENTS.md (2026). Multi-agent coordination patterns; AGENTS.md as orchestration manifest; subagent model routing via `agents.default_subagent_model`.
