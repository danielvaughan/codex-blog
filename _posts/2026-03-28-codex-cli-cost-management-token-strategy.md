---
title: "Codex CLI Cost Management: Token Strategy, Model Routing and Quota Control"
date: 2026-03-28T09:00:00+00:00
description: "How to estimate, control, and monitor Codex CLI costs for individuals and teams. Includes worked cost projections, profile routing patterns, hook-based monitoring, and enterprise governance."
substack_status: draft
tags:
  - models
  - cost-optimization
  - model-selection
  - enterprise
  - config-toml
  - cost-management
  - token-optimization
  - model-routing
---
![Sketchnote diagram for: Codex CLI Cost Management: Token Strategy, Model Routing and Quota Control](/sketchnotes/articles/2026-03-28-codex-cli-cost-management-token-strategy.png)

# Codex CLI Cost Management: Token Strategy, Model Routing and Quota Control

*Published: 2026-03-28*

The biggest surprise in Codex deployments isn't the cost of output tokens — it's the accumulated cost of conversation history. A session that reads ten files and runs ten tool calls adds 5,000+ tokens to the context on every subsequent API call. In a twenty-call session that's 100,000 extra tokens of history on top of actual output. This is what makes `/compact` a financial decision as well as a quality one.

Every prompt sent through Codex CLI burns tokens — input tokens for context, output tokens for generated code, and (for reasoning models) internal chain-of-thought tokens that never appear on screen but still hit the bill. Understanding exactly where those tokens go is the difference between a $20/month hobby budget and an eye-watering invoice.

---

## Subscription vs API: Two Different Products

Codex CLI supports two fundamentally different billing paths:

**ChatGPT Plus/Pro subscribers:** Codex quota is expressed in compute-equivalent units, resets on a **rolling 5-hour window** (not a monthly cycle), and doesn't roll over. Subscription quota is sufficient for individual exploratory use; teams generating real output typically exhaust per-user allocations.

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

The choice between billing modes matters. For predictable daily usage, subscriptions are almost always cheaper. For sporadic heavy bursts or CI/CD automation, API keys give uncapped throughput without waiting for message limits to reset.

---

## Subscription Plans and Codex Limits

As of April 2026, six ChatGPT tiers include Codex access [^sub1] [^sub2]:

| Plan | Monthly Cost | Local Messages (5 hr) | Cloud Tasks (5 hr) | Code Reviews/Week |
|------|-------------|----------------------|--------------------|--------------------|
| **Free** | $0 | Limited | — | — |
| **Go** | $8 | Limited | — | — |
| **Plus** | $20 | 33–168 (GPT-5.4) / 45–225 (5.3-Codex) | Limited | 10–25 |
| **Pro** | $200 | 223–1,120 (GPT-5.4) / 300–1,500 (5.3-Codex) | 50–400 | 100–250 |
| **Business** | $25/user | 15–60 (GPT-5.4) / 20–90 (5.3-Codex) | 5–40 | 15–30 |
| **Enterprise** | Custom | Custom | Custom | Custom |

The ranges reflect message complexity: a simple "add a docstring" costs fewer credits than a multi-file refactor. OpenAI approximates GPT-5.4 local messages at ~7 credits each, GPT-5.3-Codex at ~5 credits, and GPT-5.4-mini at ~2 credits. Cloud tasks average ~34 credits and code reviews ~25 credits [^sub1].

### Subscription Tier Break-Even Analysis

| Tier | Monthly cost | Best for | API equivalent break-even |
|---|---|---|---|
| ChatGPT Plus | $20/user | Individuals, exploratory use | ~50–80 hours/month intensive coding |
| ChatGPT Team | $25–30/user | Small teams (3–15 devs) | ~60–90 hours/user/month |
| ChatGPT Pro | $200/user | Power users, agentic workflows | ~200+ hours/month or heavy reasoning model use |
| API key mode | Pay-per-token | CI/CD pipelines, automation | N/A — predictable cost at scale |
| Enterprise | Custom pricing | 50+ devs, compliance requirements | Custom |

The break-even calculation for **Pro vs API key** hinges on reasoning model usage. Regular use of `o3`-class models — roughly 5x the token cost of standard models — means a Pro subscription's flat-rate access can be significantly cheaper than equivalent API billing at that tier [^be1].

### When API Key Mode Wins

API key mode is the right choice when:

1. **CI/CD pipelines** where usage is predictable and billing separation per team/repo is required
2. **Subagent workers on `gpt-4.1-mini`** — the cheapest path when cost can be instrumented exactly
3. **Enterprise multi-project billing** where chargeback per team is required

The trap to avoid: using API key mode without `max_tokens_per_session` configured. A single runaway session debugging a large legacy codebase can consume 500K+ tokens — roughly $8–15 in one sitting.

---

## API Token Pricing by Model

When using API key mode, billing is per million tokens. Current rates as of April 2026 [^api1] [^api2]:

| Model | Input (per 1M) | Cached Input (per 1M) | Output (per 1M) | Context Window |
|-------|----------------|----------------------|------------------|----------------|
| **GPT-5.4** | $2.50 | $1.25 | $10.00 | 256K |
| **GPT-5.4-mini** | $0.40 | $0.20 | $1.60 | 128K |
| **GPT-5.3-Codex** | $1.75 | $0.875 | $14.00 | 256K |
| **codex-mini-latest** | $1.50 | $0.75 | $6.00 | — |
| **o3** | $2.00 | — | $8.00 | — |
| **o4-mini** | $1.10 | — | $4.40 | — |

### Reasoning Tokens: The Hidden Cost

Models like o3 and o4-mini generate internal reasoning tokens as part of chain-of-thought processing. These tokens never appear in the CLI output, but they are billed as output tokens [^rt1]. A task that produces 500 visible output tokens might generate 3,000–5,000 reasoning tokens internally, multiplying the effective output cost by 5–10x.

This is why reasoning model tasks can be surprisingly expensive. Use `codex --status` mid-session to monitor cumulative token consumption.

### Cached Input Tokens

Codex CLI benefits significantly from prompt caching. When repeated or overlapping context is sent (common during iterative development), OpenAI caches the prompt prefix and charges cached input tokens at 50% of the standard input rate [^api1]. On a long session refining the same file, caching can reduce input costs by 30–40%.

Structure workflows to benefit from cached inputs: work iteratively on the same files rather than jumping between unrelated areas of the codebase.

---

## The 4x Token Efficiency Advantage

OpenAI-cited benchmarks and independent developer testing consistently show Codex CLI uses **approximately 4x fewer tokens** than Claude Code for equivalent coding tasks [^eff1].

The raw numbers from a Figma-to-code cloning benchmark:

- Claude Code: ~6.2 million tokens
- Codex CLI: ~1.5 million tokens

For a focused TypeScript task:

- Claude Code: 234,772 tokens
- Codex CLI: 72,579 tokens [^eff2]

This gap has concrete financial implications. At API rates, the effective cost of a task through Claude Code can be **four times higher** even before accounting for Claude's higher per-token price.

Claude Code's higher token consumption is not waste — it reflects a different philosophy: Claude reasons aloud, asks clarifying questions, and provides detailed explanations. This is valuable when exploring architecture or debugging complex issues. It is expensive when the desired output is already well-specified [^eff2].

The practical implication: **use each tool where its communication style adds value**. Claude Code for initial design and complex reasoning; Codex CLI for execution, refactoring, and the bulk of implementation work.

---

## The Spark Model: Turbo Worker Economics

GPT-5.3-Codex-Spark runs on Cerebras WSE-3 hardware at 1,000+ tokens per second — a 15x speed improvement over GPT-5-Codex [^spark1]. During the current research preview period, Spark usage has **separate model-specific limits and does not count against standard Codex quota** [^spark2].

This makes Spark an unusual opportunity: during the preview, it is effectively free from the normal credit budget.

### Credits-Per-Task Routing Matrix

Approximate relative credit cost by task type:

| Task type | Recommended model | Relative cost |
|---|---|---|
| File navigation, search | `gpt-5.4-mini` | 0.2x |
| Test scaffolding | `gpt-5.4-mini` | 0.2x |
| Documentation generation | `gpt-5.4-mini` | 0.2x |
| Draft implementation | `gpt-5.3-codex-spark`* | ~0x (separate pool) |
| Iterative refinement | `gpt-5.3-codex-spark`* | ~0x (separate pool) |
| Multi-file refactoring | `gpt-5.4` | 1x |
| Complex debugging | `gpt-5.4` | 1x |
| Security audit | `o4-mini` | ~3x |
| Novel algorithm design | `o3` | ~15x |

*\*Research preview, Pro tier only, separate limits* [^spark2]

---

## Worked Cost Examples

### Example 1: Generate a CRUD Endpoint

A straightforward "generate a REST endpoint for users with CRUD operations" prompt:

- **Input:** ~2,000 tokens (prompt + file context)
- **Output:** ~4,000 tokens (generated code)

| Model | Input Cost | Output Cost | **Total** |
|-------|-----------|-------------|-----------|
| GPT-5.4 | $0.005 | $0.040 | **$0.045** |
| GPT-5.4-mini | $0.001 | $0.006 | **$0.007** |

Using GPT-5.4-mini for routine scaffolding saves ~85% per request.

### Example 2: Full-Day Intensive Coding Session

Approximately 8 hours of active development — refactoring, writing tests, reviewing PRs:

- **Input:** ~200,000 tokens (cumulative context across turns)
- **Output:** ~150,000 tokens (generated code and explanations)

| Model | Input Cost | Output Cost | **Total** |
|-------|-----------|-------------|-----------|
| GPT-5.4 | $0.50 | $1.50 | **$2.00** |
| GPT-5.4-mini | $0.08 | $0.24 | **$0.32** |

Over 22 working days, that's **$44/month on GPT-5.4** or **$7/month on GPT-5.4-mini** — both cheaper than the Plus subscription with disciplined model selection.

### Example 3: CI Pipeline Code Review

Running `codex exec review --base main` on a 500-line PR:

- **Input:** ~15,000 tokens (diff + repository context)
- **Output:** ~3,000 tokens (review comments)
- **Cost on GPT-5.4:** ~$0.07 per review

At 10 PRs/day across a team, that's roughly **$15/month** — far cheaper than dedicated review tooling.

### Example 4: Five-Engineer Team (Orchestrator/Worker Pattern)

- Each engineer runs ~15 Codex tasks/day, ~3 hours of active sessions
- Orchestrator on `gpt-4.1`: 40K input + 4K output tokens per task
- 2.5 workers per task on `gpt-4.1-mini`: 25K input + 3K output tokens each

**Per-engineer per-day:**
- Orchestrators: 15 x ($0.08 + $0.032) = **$1.68**
- Workers: 37.5 x ($0.01 + $0.0048) = **$0.55**
- Total: **$2.23/day**

**Monthly team cost (22 working days):** $2.23 x 5 x 22 = **~$245/month**

If workers ran on `gpt-4.1` instead of `gpt-4.1-mini`: ~$750/month (+205%). Model routing matters.

> **Key heuristic:** If the correct output could be specified before the agent runs, a reasoning-class model is probably unnecessary. If the agent needs to explore solution space and evaluate trade-offs, reasoning models earn their cost.

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

## Credit-Based Billing for Business and Enterprise

Since April 2026, Business and Enterprise customers use a token-based credit system rather than message counting [^sub1]:

| Model | Input (cr/1M) | Cached Input (cr/1M) | Output (cr/1M) |
|-------|--------------|---------------------|----------------|
| GPT-5.4 | 62.50 | 6.25 | 375.00 |
| GPT-5.4-mini | 18.75 | 1.875 | 113.00 |
| GPT-5.3-Codex | 43.75 | 4.375 | 350.00 |

Credits are purchased in bulk and consumed based on actual token usage. This gives organisations transparent, auditable billing that maps directly to API pricing conventions.

---

## Cloud Tasks Bill Differently

Cloud tasks (submitted via `codex cloud exec` or `@Codex` in Slack) run in isolated cloud VMs and consume more resources than local sessions [^sub1]. Key differences:

- **Higher credit cost:** ~34 credits per cloud task vs ~5–7 for a local message
- **VM overhead:** cloud tasks include compute costs for the sandboxed environment
- **No caching benefit:** each cloud task starts with a fresh context, so cached input token discounts do not apply

For cost-sensitive workflows, prefer local execution and reserve cloud tasks for fire-and-forget automation where the convenience justifies the premium.

---

## The 5-Hour Window: Managing Burst Consumption

ChatGPT subscription Codex limits reset on a **rolling 5-hour window**, not a monthly cycle [^sub1]. A session that runs multiple parallel subagents consumes credits faster than sequential work. Hitting the 5-hour window limit mid-task causes Codex to pause — it resumes when the window resets, not when the monthly cycle resets.

**Strategies for window management:**

1. **Profile-based model routing** — default profile uses `gpt-4.1-mini`, explicit invocation required for the full model
2. **Context compaction before subagent spawns** — running `/compact` before spawning workers reduces per-subagent context overhead
3. **Stagger spawning on long jobs** — rather than spawning 8 subagents simultaneously, stagger by 2–3 turns to avoid simultaneous large context initialisation
4. **Spark for draft-then-review cycles** — use Spark's separate pool for rapid first-draft generation, then one `gpt-5.4` call for review and correction

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

### In-Session Monitoring with /status

Run `/status` periodically during sessions to see cumulative token usage. If token consumption is exceeding expectations, switch models or narrow scope before the session runs away.

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

## Cost Optimisation Tactics

### Trim Context Window with .codexignore

Every file Codex reads counts as input tokens. A well-maintained `.codexignore` can cut input tokens by 40–60% on large repositories [^opt1]:

```bash
# .codexignore — exclude bulky directories
node_modules/
dist/
*.lock
__pycache__/
.git/
```

### Minimise MCP Server Overhead

Each enabled MCP server injects tool definitions into the system prompt. Disable servers not actively in use:

```toml
# Only enable what you need per-project
[mcp_servers.github]
enabled = true

[mcp_servers.slack]
enabled = false
```

Fewer MCP servers = fewer tokens per turn.

### Use Concise, Nested AGENTS.md Files

Trim AGENTS.md to the essentials — every byte adds to context on every API call. Nested `AGENTS.md` files in subdirectories provide targeted context rather than loading a monolithic project-wide instruction file. Keep each under 2 KiB where possible [^opt1].

### Subagent Delegation

Subagent delegation naturally limits main-session context growth. Use it for file-heavy operations to avoid ballooning the orchestrator's context window.

---

## Enterprise Billing Attribution

Use separate API keys per team to get natural billing breakdown in the OpenAI dashboard. Configure via:

```toml
# Per-team config distributed via configuration management
api_key_source = "environment"  # reads OPENAI_API_KEY from env
```

Import OpenAI API costs into cloud cost management tools (AWS Cost Explorer, GCP Billing) for chargeback. The `postTaskComplete` hook generates per-session data; a daily aggregation job feeds your cost management system.

---

## Subscription Upgrade Decision Framework

The key insight: **Pro's value is front-loaded towards heavy reasoning model users and teams hitting the 5-hour window daily**. For developers using primarily standard models on exploratory personal projects, Plus is typically sufficient.

| Scenario | Recommended Billing | Estimated Monthly Cost |
|----------|--------------------|-----------------------|
| Casual (< 2 hrs/day) | Plus | $20 |
| Heavy individual (4+ hrs/day) | Pro | $200 |
| Light API automation | API Key + GPT-5.4-mini | $5–15 |
| Team (5 devs, moderate use) | Business | $125–150 |
| CI/CD pipeline reviews | API Key + GPT-5.4 | $15–30 |

---

## Key Numbers to Know

- `gpt-4.1-mini` is ~1/5th the cost of `gpt-4.1` for agentic sessions
- A five-engineer orchestrator/worker team: ~$245/month on API billing
- Switching workers from `mini` to full: 3x total cost increase
- Manual `/compact` at 60% context: 30–50% cost reduction per subsequent call
- `max_tokens_per_session = 200000` is a safe default ceiling for individual devs
- Codex CLI uses **~4x fewer tokens** than Claude Code for equivalent tasks [^eff1]
- GPT-5.3-Codex-Spark: **separate credit pool** during research preview — currently free from standard quota [^spark2]
- Break-even for Plus ($20) vs API key: **~50–80 hours/month** of intensive coding
- Break-even for Pro ($200) vs API key: primarily justified by **daily reasoning model (o3/o4-mini) usage**
- Local tasks: consistently cheaper than cloud tasks (PR review, Slack/Linear triggers)
- A `gpt-5.4` session in API mode consuming 500K tokens: approximately **$8–15** without `max_tokens_per_session` guardrails
- Full-day session on GPT-5.4-mini: **~$0.32/day** ($7/month over 22 days)

---

## Citations

[^sub1]: [Codex Pricing -- OpenAI Developers](https://developers.openai.com/codex/pricing) -- official pricing page with credit rate card and local vs cloud task distinction
[^sub2]: [ChatGPT Plans -- OpenAI](https://chatgpt.com/pricing/) -- subscription tier pricing
[^be1]: [OpenAI Codex Pricing 2026: API Costs, Token Limits -- Flowith Blog](https://flowith.io/blog/openai-codex-pricing-2026-api-costs-token-limits/) -- break-even analysis across subscription tiers
[^api1]: [OpenAI API Pricing](https://developers.openai.com/api/docs/pricing) -- per-token API rates
[^api2]: [OpenAI Codex Pricing 2026: API Costs, Token Limits -- Flowith Blog](https://flowith.io/blog/openai-codex-pricing-2026-api-costs-token-limits/) -- model pricing tables
[^rt1]: [OpenAI o3 Pricing: Every Plan and API Cost Explained -- PanelsAI](https://panelsai.com/openai-o3-pricing/) -- reasoning token billing
[^eff1]: [Codex vs Claude Code (2026): Benchmarks, Agent Teams & Limits Compared -- MorphLLM](https://www.morphllm.com/comparisons/codex-vs-claude-code) -- 4x token efficiency benchmark comparison
[^eff2]: [Claude Code vs Codex CLI 2026 -- NxCode](https://www.nxcode.io/resources/news/claude-code-vs-codex-cli-terminal-coding-comparison-2026) -- token count comparison on identical TypeScript tasks
[^spark1]: [Introducing GPT-5.3-Codex-Spark -- OpenAI](https://openai.com/index/introducing-gpt-5-3-codex-spark/) -- Spark model announcement, Cerebras WSE-3, 1,000+ tokens/sec
[^spark2]: [Models -- Codex Developers](https://developers.openai.com/codex/models) -- Spark separate limit pool during research preview
[^opt1]: [How to Reduce Codex CLI Token Usage: 7 Proven Optimization Strategies -- BSWEN](https://docs.bswen.com/blog/2026-03-02-reduce-codex-cli-token-usage/) -- context trimming techniques

*Source: OpenAI Codex developer documentation + book chapter research, 2026-03-28. Updated 2026-04-08 with billing mechanics and worked examples.*
