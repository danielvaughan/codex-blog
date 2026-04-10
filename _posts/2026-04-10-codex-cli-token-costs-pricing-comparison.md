---
title: "The Real Cost of AI Coding Agents: Codex CLI Token Economics vs Cursor, Copilot, Claude Code, and Windsurf"
date: 2026-04-10T15:30:00+01:00
parent: "Articles"
nav_order: 240
tags:
  - pricing
  - token-economics
  - cost-comparison
  - codex-cli
  - cursor
  - copilot
  - claude-code
  - windsurf
  - model-selection
  - enterprise
---

![Sketchnote diagram for: The Real Cost of AI Coding Agents: Codex CLI Token Economics vs Cursor, Copilot, Claude Code, and Windsurf](/sketchnotes/articles/2026-04-10-codex-cli-token-costs-pricing-comparison.png)

# The Real Cost of AI Coding Agents: Codex CLI Token Economics vs Cursor, Copilot, Claude Code, and Windsurf



---

Every AI coding agent runs on tokens, and tokens cost money. But comparing costs across tools is not straightforward: some charge per subscription, some per token, some per credit, and some use opaque "premium request" budgets that resist direct comparison. This article breaks down the real token economics of Codex CLI across every plan tier, compares them against Cursor, GitHub Copilot, Claude Code, and Windsurf, and shows where each tool's pricing model creates advantages or traps.

## OpenAI Codex CLI: Subscription Tiers and Token Rates

Codex CLI is an open-source tool that runs locally. The cost comes from the models it calls, which are gated by your ChatGPT subscription tier or billed directly through the OpenAI API [^1].

### Subscription Plans

| Plan | Price | Codex CLI Access | Cloud Codex Tasks |
|------|-------|-----------------|-------------------|
| Free | $0/mo | No | No |
| Go | $8/mo | No | No |
| Plus | $20/mo | Yes — rate-limited | Yes — limited |
| Pro 5× | $100/mo | Yes — 5× Plus limits | Yes — 5× limits |
| Pro 20× | $200/mo | Yes — 20× Plus limits | Yes — 20× limits |
| Business | Pay-as-you-go | Yes — token-based | Yes — token-based |
| Enterprise | Custom | Yes — custom limits | Yes — custom |

### Usage Limits (Per 5-Hour Window)

| Model | Plus | Pro 5× | Pro 20× |
|-------|------|--------|---------|
| GPT-5.4 (local) | 20–100 messages | 100–500 | 400–2,000 |
| GPT-5.4-mini (local) | 60–350 messages | 300–1,750 | 1,200–7,000 |
| GPT-5.3-Codex (local) | 30–150 messages | 150–750 | 600–3,000 |
| GPT-5.3-Codex (cloud) | 10–60 tasks | 50–300 | 200–1,200 |

The ranges reflect dynamic rate limits that adjust based on system load [^1].

### Credit Consumption Per Task

Each model consumes credits differently for local versus cloud tasks:

| Model | Local Task | Cloud Task |
|-------|-----------|------------|
| GPT-5.4 | ~7 credits | ~34 credits |
| GPT-5.3-Codex | ~5 credits | ~25 credits |
| GPT-5.4-mini | ~2 credits | N/A |

Fast mode doubles credit consumption across all models [^1].

### API Token Rates (Pay-As-You-Go)

For Business plan users or anyone using the API directly, per-million-token pricing applies [^2]:

| Model | Input | Cached Input | Output |
|-------|-------|-------------|--------|
| GPT-5.4 | $2.50 | $0.25 | $15.00 |
| GPT-5.4 (long context) | $5.00 | $0.50 | $22.50 |
| GPT-5.4-mini | $0.75 | $0.075 | $4.50 |
| GPT-5.4-nano | $0.20 | $0.02 | $1.25 |
| GPT-5.3-Codex | $1.75 | $0.175 | $14.00 |
| GPT-5.3-Codex (priority) | $3.50 | $0.35 | $28.00 |
| GPT-5.4-pro | $30.00 | — | $180.00 |

The 90% cached input discount is the most important number in this table. Codex CLI's prefix caching means that in a typical session, 60-80% of input tokens hit the cache. A session that appears to cost $2.50/M input tokens effectively costs $0.50-1.00/M when caching is factored in [^3].

## The Competition: What You Pay Elsewhere

### Cursor

| Plan | Price | Model Access | Usage |
|------|-------|-------------|-------|
| Hobby | Free | Limited models | Limited agent + tab completions |
| Pro | $20/mo | Frontier models (GPT-5.4, Claude, Gemini) | Extended agent limits |
| Pro+ | $60/mo | Same models | 3× usage on all models |
| Ultra | $200/mo | Same models | 20× usage, priority features |
| Teams | $40/user/mo | Same models | Shared chats, SSO, analytics |
| Enterprise | Custom | Same models | Pooled usage, SCIM, admin controls |

Cursor's pricing is opaque by design — "extended limits" and "3× usage" give no concrete token budgets. The advantage is model flexibility: Cursor routes to GPT-5.4, Claude Sonnet 4.6, or Gemini depending on the task, and the subscription covers all of them. The disadvantage is that you cannot predict or control costs at the token level [^4].

### GitHub Copilot

| Plan | Price | Premium Requests | Models |
|------|-------|-----------------|--------|
| Free | $0/mo | 50 chat requests/mo | Limited |
| Pro | $10/mo | 300 premium requests/mo | GPT-5.4, Claude Sonnet |
| Pro+ | $39/mo | 1,500 premium requests/mo | All models incl. Claude Opus 4, o3 |
| Business | $19/user/mo | 300 premium/user/mo | Frontier models, org management |
| Enterprise | $39/user/mo | 1,500 premium/user/mo | Fine-tuning, knowledge base, compliance |

Copilot uses "premium requests" as its unit of consumption. A premium request is a single interaction with a frontier model — not a token count but a request count. Simple completions cost one request; complex multi-step agent operations may cost more. The Pro tier at $10/month is the cheapest entry point to frontier model access across any tool in this comparison [^5].

### Claude Code (Anthropic)

| Plan | Price | Usage | Models |
|------|-------|-------|--------|
| Pro | $20/mo | Base rate limits | Claude Sonnet 4.6 |
| Max 5× | $100/mo | 5× Pro limits (~88K tokens/5hr) | Sonnet 4.6, Opus 4.6 |
| Max 20× | $200/mo | 20× Pro limits (~220K tokens/5hr) | Sonnet 4.6, Opus 4.6 |
| Team | Per-seat pricing | Pooled usage | All models, admin |
| Enterprise | Custom | Custom | All models, SSO, compliance |

Claude Code's API rates per million tokens [^6]:

| Model | Input | Output |
|-------|-------|--------|
| Claude Opus 4.6 | $5.00 | $25.00 |
| Claude Sonnet 4.6 | $3.00 | $15.00 |
| Claude Haiku 4.5 | $1.00 | $5.00 |

Anthropic recently simplified pricing: the full 1M-token context window is now at standard rates for Opus 4.6 and Sonnet 4.6, eliminating the long-context surcharge that previously doubled input costs beyond 200K tokens [^6]. Prompt caching (90% savings) and batch API (50% off) can stack for up to 95% cost reduction.

### Windsurf (formerly Codeium)

| Plan | Price | Credits/Month | Models |
|------|-------|--------------|--------|
| Free | $0/mo | 25 credits | Basic models |
| Pro | $15/mo | 500 credits | All premium models, SWE-1.5 |
| Teams | $30/user/mo | 500 credits/user | Admin dashboard, priority support |
| Enterprise | $60/user/mo | 1,000+ credits/user | SSO, RBAC, hybrid deployment |

Windsurf uses a credit system where different operations consume different credit amounts. Monthly credits expire; purchased add-on credits ($10 for 250) do not. At $15/month for 500 credits with frontier model access, Windsurf is the second cheapest entry point after Copilot Free [^7].

## Head-to-Head: What Does a Real Workday Cost?

Abstract pricing tables obscure the practical question: what does a day of serious coding actually cost? Consider a developer who runs 15-20 substantial agent interactions per day — not tab completions, but multi-step tasks involving file reads, edits, test runs, and iterations.

### Scenario: 20 Substantial Tasks Per Day

| Tool | Plan | Daily Cost | Monthly Cost | Notes |
|------|------|-----------|-------------|-------|
| Codex CLI | Plus ($20/mo) | ~$0.67 | $20 | May hit rate limits on heavy days |
| Codex CLI | Pro 5× ($100/mo) | ~$3.33 | $100 | Comfortable headroom |
| Codex CLI | API direct | ~$2-8 | $40-160 | Varies by model; caching helps |
| Cursor | Pro ($20/mo) | ~$0.67 | $20 | May hit "extended limits" |
| Cursor | Ultra ($200/mo) | ~$6.67 | $200 | 20× usage, priority access |
| GitHub Copilot | Pro ($10/mo) | ~$0.33 | $10 | 300 requests/mo ≈ 15/workday |
| GitHub Copilot | Pro+ ($39/mo) | ~$1.30 | $39 | 1,500 requests/mo ≈ 75/workday |
| Claude Code | Pro ($20/mo) | ~$0.67 | $20 | May hit rate limits |
| Claude Code | Max 20× ($200/mo) | ~$6.67 | $200 | ~220K tokens/5hr window |
| Windsurf | Pro ($15/mo) | ~$0.50 | $15 | 500 credits/mo; may exhaust |

### The API Direct Advantage

Codex CLI's unique position is that it is open-source and supports direct API billing. No other tool in this comparison offers this. If you exceed your subscription limits or need predictable per-token billing, you can bypass the subscription entirely and pay OpenAI API rates directly [^2]:

**A typical 30-minute Codex CLI session** consumes roughly:
- 50,000-100,000 input tokens (with 70% cache hit rate → effective cost: 15,000-30,000 uncached + 35,000-70,000 cached)
- 10,000-30,000 output tokens

**Cost using GPT-5.4 API rates:**
- Input (uncached): 30K × $2.50/M = $0.075
- Input (cached): 70K × $0.25/M = $0.018
- Output: 20K × $15.00/M = $0.30
- **Total per session: ~$0.39**

At 4-6 sessions per day, that is $1.56-2.34/day or **$31-47/month** — comparable to the $20 Plus subscription but with no rate limits and full control over model selection.

**Cost using GPT-5.4-mini API rates:**
- Input (uncached): 30K × $0.75/M = $0.023
- Input (cached): 70K × $0.075/M = $0.005
- Output: 20K × $4.50/M = $0.09
- **Total per session: ~$0.12**

At 4-6 sessions per day: **$10-14/month**. Cheaper than every subscription except Copilot Free.

### The Model Selection Lever

Codex CLI's `config.toml` profile system lets you switch models per task, which is the most effective cost optimisation available:

```toml
[profiles.default]
model = "gpt-5.4-mini"          # $0.75/$4.50 — good for most tasks

[profiles.complex]
model = "gpt-5.4"               # $2.50/$15.00 — for hard problems

[profiles.bulk]
model = "gpt-5.4-nano"          # $0.20/$1.25 — for batch/scripted work
```

A developer who uses mini for 70% of tasks, full 5.4 for 25%, and nano for 5% pays roughly 60% less than one who uses GPT-5.4 for everything [^3].

## Enterprise Cost Comparison

For a team of 50 developers:

| Tool | Plan | Monthly Cost | Annual Cost |
|------|------|-------------|-------------|
| Codex CLI | Plus (all users) | $1,000 | $12,000 |
| Codex CLI | Pro 5× (all users) | $5,000 | $60,000 |
| Codex CLI | Business (API) | $2,000-8,000 | $24,000-96,000 |
| Cursor | Teams | $2,000 | $24,000 |
| Cursor | Enterprise | Custom | Custom |
| GitHub Copilot | Business | $950 | $11,400 |
| GitHub Copilot | Enterprise | $1,950 | $23,400 |
| Claude Code | Max 5× (all users) | $5,000 | $60,000 |
| Windsurf | Teams | $1,500 | $18,000 |
| Windsurf | Enterprise | $3,000 | $36,000 |

GitHub Copilot Business at $19/user/month is the cheapest enterprise option. Codex CLI Plus at $20/user is close but lacks centralised management. For teams needing heavy usage with admin controls, Cursor Teams ($40/user) and Windsurf Teams ($30/user) sit in the middle. The most expensive options — Codex CLI Pro 20× and Claude Code Max 20× at $200/user — are for power users who exhaust lower-tier limits daily.

## Hidden Costs and Gotchas

**Codex CLI cloud tasks cost 5× local tasks.** A GPT-5.3-Codex local task costs ~5 credits; the same task on Codex Cloud costs ~25 credits. Running everything in the cloud burns your budget five times faster. Use local CLI for development work and reserve cloud tasks for CI/CD and scheduled automation [^1].

**Fast mode doubles consumption.** Codex CLI's fast mode provides lower-latency responses but consumes twice the credits. It is useful for interactive exploration but expensive for batch work.

**Cursor's opacity is a feature and a bug.** The "3×" and "20×" multipliers sound generous, but without knowing the base rate in tokens, you cannot calculate cost-per-task. Some users report hitting limits mid-afternoon on Pro; others report the same plan lasting all day. The variance depends on task complexity, which Cursor does not expose.

**Copilot's "premium requests" conflate simple and complex work.** A one-line completion and a multi-file refactoring both cost one premium request. Heavy agent usage exhausts the 300/month Pro budget in two weeks; light completion usage barely dents it.

**Claude Code's 5-hour windows reset independently.** The ~88K tokens/5hr on Max 5× is generous for focused work but can be exhausted by a single large codebase exploration. The weekly limits add a second constraint that is harder to plan around.

**Windsurf credits do not roll over.** If you have a light month, you lose unused credits. The add-on credits ($10/250) are the hedge, but the per-credit cost is higher than the subscription rate.

## Recommendations by Use Case

**Solo developer, cost-sensitive:** GitHub Copilot Pro ($10/mo) for completions + Codex CLI on Plus ($20/mo) for agent tasks. Total: $30/month for frontier model access across both workflows.

**Solo developer, heavy usage:** Codex CLI on API direct with GPT-5.4-mini as default, GPT-5.4 for complex tasks. Estimated $30-60/month with no rate limits and full model control.

**Team of 10-50, needs admin controls:** GitHub Copilot Business ($19/user) for IDE completions + Codex CLI Business (API) for agent tasks. The combination gives centralised billing, audit logs, and the full agent workflow at moderate cost.

**Power user, cost is secondary:** Codex CLI Pro 20× ($200/mo) or Claude Code Max 20× ($200/mo). Both provide the highest usage limits for their respective model ecosystems. Choose based on whether you prefer GPT-5.4 or Claude Sonnet 4.6 as your primary model.

**Enterprise with compliance requirements:** GitHub Copilot Enterprise ($39/user) for fine-tuning and knowledge base, plus Codex CLI Enterprise for agent workflows with custom limits and SSO. Budget $60-80/user/month total.

---

## Key Takeaways

- Codex CLI is the only major AI coding agent that supports direct API billing, giving power users token-level cost control and no rate limits.
- GPT-5.4 costs $2.50/$15.00 per million tokens (input/output), but prefix caching reduces effective input costs by 70-90% in typical sessions.
- GPT-5.4-mini at $0.75/$4.50 is the sweet spot for most tasks — 70% of the capability at 30% of the cost.
- GitHub Copilot Pro at $10/month is the cheapest frontier model entry point; Windsurf Pro at $15/month is second.
- At the $200/month tier, Codex CLI Pro 20× and Claude Code Max 20× offer comparable top-end usage for their respective model families.
- Cloud Codex tasks cost 5× local tasks — run locally for development, reserve cloud for CI/CD.
- For teams, the combination of Copilot Business ($19/user) for completions and Codex CLI for agent tasks provides the best cost-to-capability ratio.

---

## Citations

[^1]: Codex Rate Card — OpenAI Help Center. Subscription tiers, usage limits per 5-hour window, credit consumption per model and task type. <https://help.openai.com/en/articles/20001106-codex-rate-card>

[^2]: OpenAI API Pricing — Per-million-token rates for GPT-5.4, GPT-5.4-mini, GPT-5.4-nano, GPT-5.3-Codex, and all other models. <https://developers.openai.com/api/docs/pricing>

[^3]: Codex CLI Model Selection and Cost Optimisation — Profile-based model switching and caching economics. <https://codex.danielvaughan.com/2026/03/26/codex-cli-model-selection/>

[^4]: Cursor Pricing — Plans and pricing for Hobby, Pro, Pro+, Ultra, Teams, and Enterprise tiers. <https://www.cursor.com/pricing>

[^5]: GitHub Copilot Plans — Free, Pro, Pro+, Business, and Enterprise tiers with premium request limits and model access. <https://github.com/features/copilot/plans>

[^6]: Anthropic Claude API Pricing — Per-million-token rates for Claude Opus 4.6, Sonnet 4.6, and Haiku 4.5 with prompt caching and batch discounts. <https://platform.claude.com/docs/en/about-claude/pricing>

[^7]: Windsurf Pricing — Credit-based plans from Free to Enterprise with add-on credit purchasing. <https://windsurf.com/pricing>
