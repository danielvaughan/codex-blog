---
title: "Subscription vs API Key: What Heavy Users Actually Consume and What It Costs on Codex CLI"
date: 2026-04-10T18:50:00+01:00
tags:
  - pricing
  - api-key
  - subscription
  - token-economics
  - heavy-usage
  - cost-modelling
  - business-plan
  - codex-only-seats
---

![Sketchnote diagram for: Subscription vs API Key: What Heavy Users Actually Consume and What It Costs on Codex CLI](/sketchnotes/articles/2026-04-10-codex-cli-subscription-vs-api-key-billion-token-analysis.png)

# Subscription vs API Key: What Heavy Users Actually Consume and What It Costs on Codex CLI

Every Codex CLI user faces the same fork: authenticate with your ChatGPT subscription (Plus, Pro, Business) or plug in an OpenAI API key and pay per token. At light usage, the subscription wins on simplicity. At heavy usage, the economics diverge dramatically — and the right choice depends on how much you actually consume.

This article grounds that decision in **real usage data** from Codex CLI community reports, developer forums, and industry-wide agentic coding tool studies. It then models costs across every billing path available in April 2026 and answers: for each type of user, what does the subscription cost versus the API — and when does the API key become the only viable option?

## The Billing Paths

As of April 2026, there are four ways to pay for Codex CLI usage[^1][^2]:

| Path | How It Works | Rate Limits | Billing |
|------|-------------|-------------|---------|
| **Subscription (Plus/Pro)** | Authenticate via ChatGPT account | Rolling 5-hour windows, dynamic limits | Flat monthly fee |
| **Business Standard Seat** | ChatGPT Business workspace | Plan limits + optional credit top-ups | $20/seat/month + credits |
| **Business Codex-Only Seat** | Codex access only, no ChatGPT | No rate limits | $0/seat + token consumption |
| **API Key (Direct)** | OpenAI API key in `config.toml` | Standard API rate limits only | Per-token, pay-as-you-go |

## What Users Actually Consume: The Data

Before modelling costs, we need to establish what developers actually use. The figures below are drawn from Codex CLI community reports, usage tracking tools, and industry studies of agentic coding tools.

### Per-Session Token Consumption

| Metric | Tokens | Source |
|--------|--------|--------|
| Median context per turn | ~96K | Codex CLI usage spike analysis[^6] |
| p95 context per turn | ~200K | Same source[^6] |
| Session startup overhead | 21-22K | Up from 12-15K in earlier versions[^6] |
| Typical session (simple task) | 10K-100K | Industry agentic tool studies[^7] |
| Typical session (agent workflow) | 200K-500K | Morph AI coding cost study[^8] |
| Baseline session cost (GPT-5.4-mini rates) | $0.45-2.25 | Calculated from session tokens at $0.75/$4.50 per M[^2] |

A critical finding: **60-80% of tokens in agentic sessions are waste** — spent on finding code, re-reading context, and retrying, not on writing code[^8]. Shell tool outputs alone account for 90.3% of all tool-output characters[^6].

### Per-Developer Daily and Weekly Consumption

The table below draws from Codex CLI community reports and comparable agentic tool data. No large-scale Codex CLI usage study has been published yet, so the daily cost estimates are derived from similar agentic coding tools (primarily Claude Code) and translated to GPT-5.4-mini API rates[^2]. The relative tiers — light through extreme — are consistent across tools because they reflect developer behaviour, not tool-specific factors:

| Developer Profile | Estimated Tokens/Week | Weekly API Cost (5.4-mini) | Monthly API Cost (5.4-mini) | Source |
|-------------------|----------------------|---------------------------|----------------------------|--------|
| **Light** (1-2 sessions/day) | ~2M | ~$9 | ~$36 | Community reports, BSWEN[^6] |
| **Medium** (3-5 hours/day) | ~10M | ~$45 | ~$180 | Industry average[^7][^8] |
| **Heavy** (multi-agent workflows) | ~50M | ~$225 | ~$900 | Morph AI, community reports[^8][^10] |
| **Extreme** (documented outlier) | ~300M | ~$1,350 | ~$5,400 | Community case studies[^9] |

Data from comparable agentic coding tools (notably Claude Code, the closest published dataset) shows the **average developer spends $5-8/day** on API-rate usage, with **90% of developers staying below $12/day**[^7]. Translating these figures to GPT-5.4-mini rates, 90% of developers would consume fewer than ~5M tokens/week. These figures are proxies — Codex CLI-specific usage studies at this scale have not yet been published.

### Community-Reported Pain Points

Real Codex CLI developer reports confirm these ranges[^6][^10]:

- A single Codex CLI prompt consuming **7% of weekly Plus limits** (~600K tokens based on limit structure)
- One user exhausting **97% of weekly allowance after just three prompts**
- A Plus user burning **25% of weekly limit in 30 minutes**
- A one-line configuration change consuming **~2% of the 5-hour quota**

These reports align with the 96K median-context-per-turn figure: even "light" usage of 10-20 turns can consume 1-2M tokens.

### Token Mix Assumptions

For the cost calculations that follow, we assume this token mix (consistent across all tiers):

| Component | Share | Why |
|-----------|-------|-----|
| Input (uncached) | 24% | Fresh context, new files, first turns |
| Input (cached) | 56% | Prefix caching in sustained sessions (70% cache hit rate)[^3] |
| Output | 20% | Agent edits are targeted; prompts are long |

This 70% cache hit rate reflects Codex CLI's prefix caching in sustained sessions. Short sessions or frequent context switches reduce it to 30-50%, significantly increasing costs (see Cache Efficiency section below).

## Subscription Plan vs API Key: The Head-to-Head Comparison

This is the central question. For each type of user, what does the subscription cost versus what they would pay on an API key? The table below shows every combination.

### What Each Subscription Plan Provides

| Plan | Monthly Price | Max Tokens/Month (est.) | Effective $/M Tokens | Best For |
|------|-------------|------------------------|---------------------|----------|
| Plus | $20 | ~10M-48M | ~$0.42-2.00 | Light users |
| Pro 5× | $100 | ~48M-240M | ~$0.42 | Medium users |
| Pro 20× | $200 | ~192M-960M | ~$0.21 | Heavy users (if under ceiling) |

Plus provides 20-100 GPT-5.4 messages per 5-hour window[^1]. At ~8,000 tokens per message and 3 usable windows per day, that is 2.4M-12M tokens/week or ~10M-48M/month. Pro 5× and Pro 20× multiply these limits accordingly.

### Plan vs API Cost Per User Type

The table below compares subscription cost against API cost (GPT-5.4-mini) for each usage tier. **This is the table that answers the question.**

| User Type | Tokens/Week | Tokens/Month | Best Subscription | Sub Cost/Month | API Cost/Month (5.4-mini) | Winner | Saving |
|-----------|------------|-------------|-------------------|---------------|--------------------------|--------|--------|
| **Light** | 2M | ~8M | Plus ($20) | **$20** | $36 | Sub | $16/mo (44%) |
| **Medium** | 10M | ~40M | Plus ($20) | **$20** | $180 | Sub | $160/mo (89%) |
| **Heavy** | 50M | ~200M | Pro 20× ($200) | **$200** | $900 | Sub | $700/mo (78%) |
| **Extreme** | 300M | ~1.2B | ❌ None (ceiling exceeded) | N/A | $5,400 | API only | — |
| **Team of 10** | 100M | ~400M | Codex-only seats | $0 + tokens | $1,800 | Codex-only | Admin controls |
| **Team of 50 + CI/CD** | 1B | ~4.3B | Codex-only + Batch | $0 + tokens | $4,500-9,000 | Codex-only | Admin + batch |

**Key findings:**
- For light and medium users, **subscriptions are massively subsidised** — Plus at $20/month provides $180/month of API-equivalent usage
- For heavy users consuming up to ~200M tokens/month, **Pro 20× at $200/month still beats API rates** — you get $900 worth of API usage for $200
- The **subscription ceiling breaks at ~960M tokens/month** (~240M/week). Above that, no subscription plan has enough capacity
- **One billion tokens/week is a team-level volume** — 50 developers plus CI/CD pipelines, not an individual at a keyboard

### The Same Comparison With GPT-5.4 (Full Model)

If you use GPT-5.4 instead of 5.4-mini, the subscription subsidy is even larger — but you hit rate limits sooner because each message consumes more of the allowance:

| User Type | Tokens/Week | API Cost/Month (5.4) | API Cost/Month (5.4-mini) | Best Sub Cost | Sub Saving vs 5.4 | Sub Saving vs mini |
|-----------|------------|---------------------|--------------------------|--------------|-------------------|-------------------|
| **Light** | 2M | $120 | $36 | $20 (Plus) | $100 (83%) | $16 (44%) |
| **Medium** | 10M | $600 | $180 | $20 (Plus) | $580 (97%) | $160 (89%) |
| **Heavy** | 50M | $3,000 | $900 | $200 (Pro 20×) | $2,800 (93%) | $700 (78%) |
| **Extreme** | 300M | $18,000 | $5,400 | N/A | API only | API only |

The subsidy on Plus is staggering: a medium user on GPT-5.4 gets **$600 of API value for $20**. This is why OpenAI imposes strict rate limits — the subscription is deliberately priced well below cost.

## Model Selection: The Biggest Cost Lever on the API

Once you are past the subscription ceiling and using an API key, model selection becomes the primary cost control. The table below shows the relative cost of each model, indexed against GPT-5.4 as the baseline:

| Model | Input/Cached/Output ($/M tokens) | Weekly Cost (50M tokens) | Relative to GPT-5.4 | Plain English |
|-------|----------------------------------|-------------------------|---------------------|---------------|
| GPT-5.4-pro | $30.00/—/$180.00 | $12,300 | 6.58× | **6.6× more expensive** — reasoning-only, not for routine coding |
| GPT-5.4 (priority) | $5.00/$0.50/$30.00 | $3,740 | 2.00× | **2× baseline** — guaranteed low-latency SLA |
| GPT-5.4 | $2.50/$0.25/$15.00 | $1,870 | 1.00× | **Baseline** — frontier capability |
| GPT-5.3-Codex | $1.75/$0.175/$14.00 | $1,659 | 0.89× | **11% cheaper** — code-specialised |
| GPT-5.4-mini | $0.75/$0.075/$4.50 | $561 | 0.30× | **70% cheaper** — the sweet spot for most tasks |
| GPT-5.4-nano | $0.20/$0.02/$1.25 | $155 | 0.08× | **92% cheaper** — batch/scripted work only |

**Key insight: GPT-5.4-mini delivers ~80% of GPT-5.4's coding capability at 30% of the cost.**[^3] For a heavy user, defaulting to mini and escalating to full 5.4 only when needed is the single most impactful cost decision.

GPT-5.4-pro at $30/$180 per million tokens (input/output) has **no cached input discount**[^2]. At heavy volume it costs $12,300/week for just 50M tokens. This model is designed for complex multi-step reasoning — research problems, novel algorithm design — not routine coding tasks. Using it as a default model is a billing catastrophe.

### Blended Model Strategies

Most heavy API users should not pick a single model. Here is how different blending strategies compare at the heavy individual tier (50M tokens/week):

| Strategy | Model Mix | Monthly Cost | vs All-5.4 | vs All-mini |
|----------|----------|-------------|-----------|------------|
| All GPT-5.4 | 100% 5.4 | $7,480 | — | +233% |
| Conservative blend | 50% mini / 40% 5.4 / 10% nano | $4,584 | −39% | +104% |
| **Recommended blend** | **70% mini / 25% 5.4 / 5% nano** | **$3,472** | **−54%** | **+55%** |
| Aggressive blend | 85% mini / 10% 5.4 / 5% nano | $2,660 | −64% | +19% |
| All GPT-5.4-mini | 100% mini | $2,244 | −70% | — |
| All GPT-5.4-nano | 100% nano | $619 | −92% | −72% |
| Budget batch | 100% mini (batch/flex) | $1,122 | −85% | −50% |

A blended strategy using Codex CLI profiles:

```toml
[profiles.default]
model = "gpt-5.4-mini"          # 70% of tasks

[profiles.complex]
model = "gpt-5.4"               # 25% of tasks

[profiles.bulk]
model = "gpt-5.4-nano"          # 5% of tasks
```

### Batch and Flex Processing (50% Discount)

For non-interactive workloads (CI/CD, code review pipelines, bulk refactoring), the Batch API and Flex processing both halve costs[^2]:

| Model | Standard Monthly (50M/wk) | Batch/Flex Monthly | Savings |
|-------|--------------------------|-------------------|---------|
| GPT-5.4 | $7,480 | $3,740 | 50% |
| GPT-5.4-mini | $2,244 | $1,122 | 50% |
| GPT-5.4-nano | $619 | $310 | 50% |
| GPT-5.3-Codex | $6,636 | $3,318 | 50% |

## Business Codex-Only Seats: API Rates Without the Hassle

Codex-only seats on ChatGPT Business bill at token consumption rates with no fixed monthly fee and no rate limits[^4]. For organisations, this provides:

- **Per-user spend controls** — admins set monthly credit limits per seat or per user[^5]
- **Centralised billing** — all usage appears on the workspace invoice
- **No rate limits** — unlike Plus/Pro, consumption is not gated by 5-hour windows
- **Usage monitoring** — admins can view per-user and per-seat-type consumption in the workspace billing dashboard[^5]

The token rates for Codex-only seats align with standard API pricing[^4]. For a team, this means the same costs as API key billing but with enterprise controls layered on top.

### Standard Business Seats and Codex CLI

A Standard Business seat ($20/month) includes Codex CLI access with the same rate limits as Plus[^4]. Critically, **Codex usage on Standard Business seats does consume from the seat's allocation and is visible to workspace admins** — admins can see per-user credit consumption and set spend limits by seat type or by individual user[^5]. This means:

- Usage is **monitored**: yes, workspace admins see Codex consumption
- Usage is **capped**: Standard seats are subject to the same 5-hour rolling window limits as Plus
- **Overage billing**: if credits are exhausted, usage stops unless the admin has enabled additional credit purchasing

For heavy users on Business plans, the recommended setup is a Codex-only seat rather than a Standard seat — it removes the rate limit ceiling and provides cleaner usage attribution.

## The Hidden Cost: Cache Efficiency

The calculations above assume a 70% cache hit rate on input tokens. This number is not guaranteed — it depends on how you use Codex CLI:

| Usage Pattern | Typical Cache Hit Rate | Impact on Monthly Cost (5.4-mini, 50M/wk) |
|---------------|----------------------|-------------------------------------------|
| Long continuous sessions (30+ min) | 75-85% | $1,960-2,100 |
| Short sessions, same codebase | 60-70% | $2,200-2,400 |
| Frequent context switches | 30-50% | $2,600-3,000 |
| Cold starts (new repos, CI/CD) | 5-15% | $3,200-3,600 |

The difference between best-case and worst-case caching is **~$1,500/month** on GPT-5.4-mini for a heavy user. For GPT-5.4, the gap widens to **~$4,800/month**.

Maximising cache hits is the second most impactful cost lever after model selection:

- **Use long sessions** rather than many short ones
- **Keep the same codebase context** across prompts within a session
- **Use profiles** to avoid switching models mid-session (model switches invalidate the cache)
- **Avoid `--no-cache`** unless debugging

## Recommendations by Usage Tier

**Light user (~2M tokens/week):**
- **Recommended:** Plus at $20/month
- **API equivalent:** ~$36/month (GPT-5.4-mini)
- **Why subscription:** 44% cheaper, simpler setup, includes ChatGPT access. You will rarely hit rate limits.

**Medium user (~10M tokens/week):**
- **Recommended:** Plus at $20/month
- **API equivalent:** ~$180/month (GPT-5.4-mini)
- **Why subscription:** 89% cheaper — the biggest subsidy of any tier. You may occasionally hit 5-hour window limits on heavy days. If that becomes frequent, Pro 5× ($100) eliminates the risk and is still 44% cheaper than API.

**Heavy individual (~50M tokens/week):**
- **Recommended:** Pro 20× at $200/month
- **API equivalent:** ~$900/month (GPT-5.4-mini) or ~$3,000/month (GPT-5.4)
- **Why subscription:** 78% cheaper than mini API rates. If you regularly exceed ~240M tokens/week, switch to an API key with GPT-5.4-mini default and a blended model strategy.

**Extreme individual (~300M tokens/week):**
- **Recommended:** API key (no subscription has enough capacity)
- **Cost:** ~$5,400/month on GPT-5.4-mini, or ~$3,472/month with the recommended 70/25/5 blend
- **Why API:** Pro 20× caps out around ~240M tokens/week. There is no subscription plan that can absorb this volume.

**Team of 10 medium users (~100M tokens/week):**
- **Recommended:** Codex-only seats on ChatGPT Business
- **Cost:** ~$1,800/month (API rates, no seat fee)
- **Why Codex-only:** Same API rates as direct API key, but with admin spend controls, per-user monitoring, and centralised billing. Compare to 10 × Pro 20× = $2,000/month — similar cost but Codex-only has no rate limits.

**Team of 50 with CI/CD (~1B tokens/week):**
- **Recommended:** Codex-only seats (interactive) + Batch API (pipelines)
- **Cost:** Interactive at mini rates ~$2,250/month + CI/CD on batch ~$2,244/month = **~$4,500/month total**
- **Why:** Far less than 50 × $200 Pro subscriptions ($10,000/month), with no rate limits and admin controls.

---

## Key Takeaways

- **Subscriptions are massively subsidised.** Plus at $20/month delivers $180/month of API-equivalent value for a medium user — an 89% subsidy. But the subsidy comes with hard volume ceilings.
- **The subscription ceiling breaks around ~240M tokens/week.** Below that, Pro 20× at $200/month beats API rates. Above it, the API key is the only option.
- **One billion tokens/week is a team number,** not an individual one — achievable by 50 developers plus CI/CD automation, not by a single person at a keyboard.
- **Model selection is the largest cost lever on the API:** GPT-5.4-mini at 30% of GPT-5.4's cost handles 70-80% of tasks. A blended strategy saves 54%.
- **60-80% of tokens in agentic sessions are waste** — spent on context re-reading and retries, not writing code[^8]. Reducing waste is as impactful as choosing a cheaper model.
- **Cache hit rate is the second largest lever:** the difference between 80% and 15% cache hits is ~$1,500/month on GPT-5.4-mini for a heavy user.
- **Codex-only seats** on ChatGPT Business provide API-rate billing with enterprise admin controls — the recommended path for teams.
- **The Batch API halves costs** for non-interactive workloads.

---

## Citations

[^1]: Codex Pricing — OpenAI Developers. Subscription tiers, usage limits per 5-hour window, Pro 5×/20× multipliers, promotional boosts. <https://developers.openai.com/codex/pricing>

[^2]: OpenAI API Pricing — Per-million-token rates for GPT-5.4, GPT-5.4-mini, GPT-5.4-nano, GPT-5.3-Codex including Standard, Batch, Flex, and Priority tiers. <https://developers.openai.com/api/docs/pricing>

[^3]: Codex CLI Model Selection and Cost Optimisation — Profile-based model switching, prefix caching economics, and cache hit rate impact on effective costs. <https://codex.danielvaughan.com/2026/03/26/codex-cli-model-selection/>

[^4]: Codex Rate Card — OpenAI Help Center. Codex-only seat billing model, token consumption rates, Standard vs Codex-only seat comparison. <https://help.openai.com/en/articles/20001106-codex-rate-card>

[^5]: Managing Credits and Spend Controls in ChatGPT Business — OpenAI Help Center. Admin controls for per-user and per-seat-type credit limits, usage monitoring dashboard. <https://help.openai.com/en/articles/20001155-managing-credits-and-spend-controls-in-chatgpt-business>

[^6]: Why Is My Codex CLI Token Usage Suddenly So High? — BSWEN (March 2026). Median context per turn (~96K), p95 (~200K), startup overhead (21-22K), shell output share (90.3%), community reports of single-prompt quota consumption. <https://docs.bswen.com/blog/2026-03-02-codex-cli-token-usage-spike/>

[^7]: Claude Code Token Limits: A Guide for Engineering Leaders — Faros.ai. Average developer spend $5-8/day, 90% under $12/day. **Note:** this data is from Claude Code (a comparable agentic coding tool); it is used here as the best available proxy for Codex CLI usage patterns. <https://www.faros.ai/blog/claude-code-token-limits>

[^8]: The Real Cost of AI Coding in 2026 — Morph. Agent session costs, 60-80% token waste rates, $500-2,000/month for heavy API users, 47-iteration agent loop case study. <https://www.morphllm.com/ai-coding-costs>

[^9]: Claude Code Pricing 2026: Plans, Token Costs, and Real Usage Estimates — Verdent Guides. Usage tiers (light $2-5/day, medium $6-12/day, heavy $20-60+/day), extreme user case study (10B tokens / 8 months = ~312M tokens/week). **Note:** Claude Code data used as proxy; extreme case figures translated to Codex CLI API rates. <https://www.verdent.ai/guides/claude-code-pricing-2026>

[^10]: Codex Usage After the Limit Reset Update — OpenAI Developer Community. Single prompt eating 7% of weekly limits, 97% weekly allowance after three prompts. <https://community.openai.com/t/codex-usage-after-the-limit-reset-update-single-prompt-eats-7-of-weekly-limits-plus-tier/1365284>
