---
title: "Codex CLI Cost Management: Token Strategy, Model Routing and Quota Control"
date: 2026-03-28
tags: [cost-management, token-optimization, model-routing, profiles, quota, enterprise]
description: "How to estimate, control, and monitor Codex CLI costs for individuals and teams. Includes worked cost projections, profile routing patterns, hook-based monitoring, and enterprise governance."
substack_status: draft
---
![Sketchnote diagram for: Codex CLI Cost Management: Token Strategy, Model Routing and Quota Control](/sketchnotes/articles/2026-03-28-codex-cli-cost-management-token-strategy.png)


# Codex CLI Cost Management: Token Strategy, Model Routing and Quota Control

*Published: 2026-03-28*

The biggest surprise in Codex deployments isn't the cost of output tokens — it's the accumulated cost of conversation history. A session that reads ten files and runs ten tool calls adds 5,000+ tokens to the context on every subsequent API call. In a twenty-call session that's 100,000 extra tokens of history on top of actual output. This is what makes `/compact` a financial decision as well as a quality one.

---

## Subscription vs API: Two Different Products

**ChatGPT Plus/Pro subscribers:** Codex quota is expressed in compute-equivalent units, resets monthly, and doesn't roll over. Subscription quota is sufficient for individual exploratory use; teams generating real output typically exhaust per-user allocations.

**API key users:** Billed per token at published rates. The current ratio that matters most:

| Model | Relative cost | Best for |
|---|---|---|
| `gpt-4.1-nano` | 0.05x | Search, formatting, deterministic transforms |
| `gpt-4.1-mini` | 0.2x | Exploration, test gen, docs, draft refactoring |
| `gpt-4.1` | 1.0x | Complex multi-file work, bug diagnosis in large codebases |
| `o4-mini` | ~0.6x (+ reasoning tokens) | Security audits, novel algorithms, deep reasoning |
| `o3` | ~5x | Reserve for hardest problems |

*Prices approximate as of March 2026 — verify at openai.com/pricing before budgeting.*

The `gpt-4.1-mini` / `gpt-4.1` cost ratio of 5x is the most important number in Codex cost management. Most agentic tasks don't require the full model.

---

## Estimating Team Costs

A worked example for a **five-engineer team, orchestrator/worker pattern**:

- Each engineer runs ~15 Codex tasks/day, ~3 hours of active sessions
- Orchestrator on `gpt-4.1`: 40K input + 4K output tokens per task
- 2.5 workers per task on `gpt-4.1-mini`: 25K input + 3K output tokens each

**Per-engineer per-day:**
- Orchestrators: 15 × ($0.08 + $0.032) = **$1.68**
- Workers: 37.5 × ($0.01 + $0.0048) = **$0.55**
- Total: **$2.23/day**

**Monthly team cost (22 working days):** $2.23 × 5 × 22 = **~$245/month**

If workers ran on `gpt-4.1` instead of `gpt-4.1-mini`: ~$750/month (+205%). Model routing matters.

> **Key heuristic:** If you could specify the correct output before the agent runs, you probably don't need a reasoning-class model. If the agent needs to explore solution space and evaluate trade-offs, reasoning models earn their cost.

---

## Three Configuration Tools

### 1. Per-Session Token Ceiling

```toml
# ~/.codex/config.toml
max_tokens_per_session = 200000
```

Hard limit on cumulative tokens (input + output) across all API calls in the session. Prevents runaway sessions. Note: counts tokens across all calls including those before `/compact`.

### 2. Named Profiles for Model Routing

```toml
[profiles.base]
sandbox_mode    = "workspace-write"
approval_policy = "on-request"

[profiles.explore]
inherits = "base"
model    = "gpt-4.1-mini"
max_tokens_per_session = 100000

[profiles.commit]
inherits = "base"
model    = "gpt-4.1"
max_tokens_per_session = 300000

[profiles.reason]
inherits = "base"
model    = "o4-mini"
max_tokens_per_session = 150000

default_profile = "explore"
```

Invoke explicitly:
```bash
codex --profile commit "implement the new user authentication flow"
codex --profile reason "audit this authentication module for security vulnerabilities"
```

The `explore` profile is the default — cheap, capped, appropriate for browsing and quick experiments.

### 3. Enterprise: requirements.toml Cost Policy

```toml
# requirements.toml — distributed centrally
[required]
max_tokens_per_session = { max = 400000 }
model = { allowed = ["gpt-4.1", "gpt-4.1-mini", "gpt-4.1-nano", "o4-mini"] }
```

Prevents engineers from accidentally running `o3` on routine tasks. Effective only if distributed and updated centrally via MDM.

---

## Monitoring with Hooks

### Local cost log (individual developer)

```toml
[[hooks]]
event   = "postTaskComplete"
command = """
jq -n --argjson data "$CODEX_HOOK_DATA" \
  '{ts: now | todate, model: $data.model, tokens_in: $data.usage.input_tokens, tokens_out: $data.usage.output_tokens, session_id: $data.session_id}' \
  >> ~/.codex/usage.jsonl
"""
```

Analyse after a week:
```bash
jq -s '[.[] | select(.ts | startswith("2026-03"))] |
  {input_cost: (map(.tokens_in) | add) * 0.000002,
   output_cost: (map(.tokens_out) | add) * 0.000008}' \
  ~/.codex/usage.jsonl
```

### Team webhook

```toml
[[hooks]]
event   = "postTaskComplete"
command = """
curl -s -X POST "$CODEX_COST_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d "$CODEX_HOOK_DATA"
"""
```

### Alert on large sessions

```toml
[[hooks]]
event   = "postTaskComplete"
command = """
TOTAL=$(echo "$CODEX_HOOK_DATA" | jq '.usage.input_tokens + .usage.output_tokens')
if [ "$TOTAL" -gt 500000 ]; then
  curl -s -X POST "$SLACK_WEBHOOK_URL" \
    -H "Content-Type: application/json" \
    -d "{\"text\": \"Large Codex session: $TOTAL tokens\"}"
fi
"""
```

Fires when a single session exceeds ~500K tokens (~$1.50–2.00 at current gpt-4.1 rates).

---

## Cost-Quality Decision Matrix

| Task | Use |
|------|-----|
| Code search and navigation | `nano` or `mini` |
| Formatting, linting | `nano` |
| Test generation (known patterns) | `mini` |
| Documentation | `mini` |
| Exploratory refactoring | `mini` |
| Complex multi-file refactoring | `gpt-4.1` |
| Architecture design | `gpt-4.1` or `o4-mini` |
| Security audit | `o4-mini` or `o3` |
| Novel algorithm | `o4-mini` or `o3` |
| Bug diagnosis in large codebase | `gpt-4.1` |

> **Tip from `ccusage`:** Run `ccusage` against your `~/.codex` session files to see per-day token consumption and identify which sessions are your biggest cost drivers.

---

## The `/compact` Command as a Financial Tool

Every subsequent API call after a file read re-charges for that file's tokens as conversation history. A long uncompacted session accumulates this debt on every turn.

Running `/compact` mid-session replaces the detailed history with a summary — typically reducing context by 30–50%. This has a direct cost benefit for every subsequent API call in that session.

**Pattern for long sessions:** `/compact` when the context meter hits ~60% full. Don't wait until compaction is forced — forced compaction under pressure loses more detail than proactive manual compaction.

---

## Enterprise Billing Attribution

Use separate API keys per team to get natural billing breakdown in the OpenAI dashboard. Configure via:

```toml
# Per-team config distributed via configuration management
api_key_source = "environment"  # reads OPENAI_API_KEY from env
```

Import OpenAI API costs into cloud cost management tools (AWS Cost Explorer, GCP Billing) for chargeback. The `postTaskComplete` hook generates per-session data; a daily aggregation job feeds your cost management system.

---

## Key Numbers to Know

- `gpt-4.1-mini` is ~1/5th the cost of `gpt-4.1` for agentic sessions
- A five-engineer orchestrator/worker team: ~$245/month on API billing
- Switching workers from `mini` to full: 3x total cost increase
- Manual `/compact` at 60% context: 30–50% cost reduction per subsequent call
- `max_tokens_per_session = 200000` is a safe default ceiling for individual devs

*Source: OpenAI Codex developer documentation + book chapter research, 2026-03-28*
