---
title: "GPT-6.1 Sol Quota Governance Playbook: Protecting Weekly Allowances at Team Scale"
parent: "Articles"
nav_order: 1226
date: 2026-11-10T17:00:00+00:00
last_modified_at: 2026-11-11T03:15:17Z
tags: ["codex-cli", "gpt-6", "sol", "quota-management", "cost-governance", "teams", "agents-md", "guardian", "configuration", "enterprise"]
---

# GPT-6.1 Sol Quota Governance Playbook: Protecting Weekly Allowances at Team Scale


---

One engineer burning 89 per cent of their weekly allowance in a single session is a personal inconvenience. Five engineers doing it on the same Monday morning is an availability incident. The Token Spiral article[^1] covers emergency mitigations for individuals. This playbook covers what engineering managers and platform teams need to put in place *before* that happens.

The November 2026 billing regression — where GPT-6.1 Terra's cached input discount was misapplied to uncached Sol tokens — amplifies a structural cost pattern that will outlast the regression itself.[^2] Sol's turns-squared token accumulation is not a bug. It is an architectural characteristic of a long-context reasoning model. Teams that treat quota governance as an afterthought will continue to hit preventable limits even after OpenAI patches the billing defect.

This playbook is organised around three control layers: **visibility**, **policy**, and **enforcement**. Each layer can be implemented independently, but they compound in effectiveness.

---

## Layer 1: Visibility — Know Before You Overspend

The first problem in every quota incident report is the same: engineers discovered the problem after the allowance was exhausted. The goal of the visibility layer is to surface usage data in real time, not in the post-mortem.

### Session-Level Diagnostics with ccusage

The `ccusage` CLI tool provides per-session and rolling-window token counts for Codex CLI sessions.[^3] Install it once per machine and configure it as a session-end hook:

```toml
# config.toml
[[hooks.postsession]]
command = "ccusage report --session $CODEX_SESSION_ID --format compact >> ~/.codex/usage.log"
```

This writes a one-line summary after every session: timestamp, model, turns, input tokens, output tokens, cached tokens, and estimated cost. For teams using shared infrastructure, redirect the log to a shared path accessible to your monitoring stack.

### Rolling Weekly Totals

Codex CLI exposes current-period usage via `codex usage`:

```bash
codex usage --period week --model gpt-6-sol --format json
```

Output includes `tokens_used`, `tokens_limit`, `percent_consumed`, and `resets_at`. Integrate this into your team dashboard using a cron job or a Guardian pretooluse hook that emits a warning to Slack when any engineer crosses 50 per cent:

```bash
# Emit warning when weekly usage exceeds 50 per cent
USAGE=$(codex usage --period week --model gpt-6-sol --format json | jq '.percent_consumed')
if (( $(echo "$USAGE > 50" | bc -l) )); then
  codex-notify slack "#eng-platform" "⚠️ $(whoami) is at ${USAGE}% of weekly Sol quota"
fi
```

### Rollout Budget Warnings

The `rollout_budget` feature in `config.toml` provides in-session alerts before quota is exhausted:

```toml
[features.rollout_budget]
enabled = true
limit_tokens = 250000
reminder_at_remaining_tokens = [100000, 50000, 20000, 10000]
```

Set `limit_tokens` to a fraction of the weekly allowance appropriate for a single session — 10–15 per cent is a reasonable default for most engineering workflows. The warnings appear inline in the TUI and do not interrupt the session, giving the engineer the choice to wrap up or continue with awareness.

---

## Layer 2: Policy — Encode Governance into AGENTS.md

Visibility surfaces the problem. Policy prevents it. The most durable place to encode model selection and cost discipline policies is `AGENTS.md` — it is version-controlled, project-scoped, and read by Codex CLI at session start.[^4]

### Model Tier Policy

Define explicit defaults and constraints for model selection:

```markdown
## Model Selection Policy

- Default interactive sessions: gpt-6-sol (Ultra mode)
- Complex multi-step planning or architecture sessions: gpt-6-astra (flag required: `--model astra`)
- Code formatting, test generation, documentation, file operations: gpt-6-luna
- Subagent spawns (all types): gpt-6-luna unless explicitly overridden
- Weekly quota threshold trigger: when >60% of weekly Sol allowance is consumed,
  switch to gpt-6-luna for all non-critical sessions until Monday reset
```

The subagent policy is particularly high-impact. A Sol parent spawning Sol subagents multiplies cost geometrically — each spawned agent starts its own context accumulation curve. Routing spawns to Luna by default captures 40–60 per cent of session cost with no quality impact on routine tasks.[^5]

### Output Discipline Policy

Verbose command output is a primary driver of context accumulation. Encoding concise output defaults in AGENTS.md prevents the most common sources of token inflation:

```markdown
## Output Discipline

Prefer concise output in all shell commands:
- `git log`: always use `--oneline`, limit to 20 entries
- Test runners: pipe to `tail -100` unless debugging a failure
- Package managers: use `--quiet` or `--silent`
- `docker compose logs`: always use `--tail 50`
- `find`: limit to `-maxdepth 3` unless deep search is explicitly needed
- Avoid `--verbose` and `--stat` flags unless specifically requested
```

### MCP Registry Policy

Large MCP registries are a stealth cost multiplier — every registered server injects its full tool schema into every turn regardless of whether the tools are used in that session.[^6] Encode per-project MCP scope in AGENTS.md:

```markdown
## MCP Tool Scope

Enable only the MCP servers relevant to this project:
- Required: filesystem, github
- Optional (enable when needed): postgres, slack
- Disabled by default: brave-search, puppeteer, notion

Run `codex mcp status` at session start to verify active servers.
```

---

## Layer 3: Enforcement — Guardian Budgets and Circuit Breakers

Policy encodes intent. Enforcement ensures it is followed. Guardian's elastic budget system provides programmatic enforcement at the session and fleet level.[^7]

### Per-Engineer Session Budgets

Configure Guardian to enforce a per-session token ceiling with a soft warning and a hard stop:

```toml
[guardian.budget]
enabled = true
per_session_limit_tokens = 300000
soft_limit_tokens = 200000
soft_limit_action = "warn_and_log"
hard_limit_action = "pause_and_confirm"
```

The `pause_and_confirm` action suspends tool execution and presents the engineer with a summary of tokens consumed so far. They can choose to continue, downgrade to Luna mode, or end the session. This is preferable to a hard stop because it preserves agency while making the cost visible at the decision point.

### Fleet-Level Budget Monitoring

For organisations managing multiple Pro seats, Guardian can aggregate session data across engineers using a shared Guardian server:

```toml
[guardian.server]
endpoint = "https://guardian.internal/fleet"
report_interval_seconds = 300
alert_webhook = "https://hooks.slack.com/services/YOUR/WEBHOOK"

[guardian.fleet_budget]
weekly_sol_limit_tokens = 5000000     # 5M tokens per team per week
alert_at_percent = [60, 80, 95]
```

When the fleet crosses 60 per cent of the weekly allocation, Guardian emits a Slack alert to the platform channel. At 80 per cent, it optionally sends individual notifications to engineers whose sessions are in the top quartile for consumption. This creates accountability without surveillance — engineers see their own numbers, not comparisons.

### Circuit Breaker for Automated Pipelines

CI/CD pipelines running Sol in non-interactive mode[^8] are particularly prone to unexpected quota consumption because they lack the natural session-length limits of human-driven sessions. Add a circuit breaker as a prerun check:

```bash
#!/bin/bash
# .github/scripts/check-quota.sh
REMAINING=$(codex usage --period week --model gpt-6-sol --format json | jq '.percent_remaining')
if (( $(echo "$REMAINING < 15" | bc -l) )); then
  echo "⛔ Sol quota below 15% — switching CI pipeline to Luna model"
  export CODEX_MODEL=gpt-6-luna
fi
```

Include this in your GitHub Actions workflow before any Codex CLI step:

```yaml
- name: Check quota before Codex run
  run: bash .github/scripts/check-quota.sh
  
- name: Run Codex
  run: codex run --model ${CODEX_MODEL:-gpt-6-sol} --non-interactive ...
```

---

## Putting It Together: Recommended Rollout Sequence

Teams that have not yet implemented any quota governance should prioritise in this order:

| Priority | Control | Effort | Impact |
|----------|---------|--------|--------|
| 1 | `rollout_budget` warnings in `config.toml` | 5 minutes | Immediate — catches runaway sessions |
| 2 | Subagent Luna routing in `AGENTS.md` | 10 minutes | 40–60% reduction in multi-agent session costs |
| 3 | Output discipline in `AGENTS.md` | 10 minutes | 20–40% reduction in context accumulation rate |
| 4 | MCP registry scoping in `AGENTS.md` | 15 minutes | 10–50% per-turn baseline reduction (varies by registry size) |
| 5 | Guardian per-session budget enforcement | 30 minutes | Hard ceiling with graceful pause |
| 6 | Fleet-level monitoring with Guardian server | 2–4 hours | Organisational visibility and early warning |

The first four items are pure configuration changes that can be committed to your project repository in a single PR. They require no infrastructure and take effect immediately at the next session start. Items 5 and 6 require Guardian server configuration and may need platform team involvement.

---

## A Note on the November Regression

OpenAI has acknowledged the billing regression affecting Terra-cached tokens being applied to uncached Sol calls.[^2] A fix is expected in the coming days. When it ships, per-session costs will return to their pre-November baseline.

The governance measures in this playbook remain relevant after the fix. Sol's turns-squared cost dynamics are structural, not a bug. The regression compressed the timeline on which teams would naturally encounter quota pressure — but the underlying dynamics were always present, and they will remain after the patch. Teams that implement these controls now will be better positioned for the full v0.169.0 stable release, which introduces additional agentic features that increase session depth and, with it, context accumulation potential.

---

[^1]: "The Token Spiral: Why GPT-6.1 Sol Is Eating Your Weekly Allowance." `articles/2026-11-10-token-spiral-gpt61-sol-weekly-quota-burn-codex-cli.md`. Published 10 November 2026.
[^2]: Community-identified billing regression: GPT-6.1 Terra cached input pricing applied to uncached Sol tokens. GitHub issue #60530, confirmed circa 5 November 2026.
[^3]: ccusage CLI tool — session-level token diagnostics for Codex CLI. Community-maintained. Available via `npm install -g ccusage`.
[^4]: AGENTS.md specification and project scope behaviour: `articles/2026-04-10-agents-md-codex-cli-project-configuration.md`.
[^5]: Subagent model routing cost analysis. See: `articles/2026-07-24-codex-cli-multi-agent-v2-governance-playbook.md`, section "Sub-agent cost implications".
[^6]: MCP tool schema injection cost: approximately 600 tokens per tool per turn. A 93-tool registry adds ~55,000 tokens per turn. Source: community benchmarking, OpenAI developer forum.
[^7]: Guardian v3 elastic budget system: `articles/2026-11-05-guardian-v3-evidence-pipeline-snapshots-cursors-housekeeping.md` and `articles/2026-11-06-guardian-v3-production-configuration-guide-codex-cli.md`.
[^8]: CI/CD non-interactive trust requirement (v0.169.0+): `articles/2026-11-09-codex-ci-cd-non-interactive-trust-explicit-project-trust-flag.md`.
