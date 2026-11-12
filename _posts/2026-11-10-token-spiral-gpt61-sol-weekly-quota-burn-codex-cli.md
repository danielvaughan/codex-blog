---
title: "The Token Spiral: Why GPT-6.1 Sol Is Eating Your Weekly Allowance"
parent: "Articles"
nav_order: 1225
date: 2026-11-10T13:00:00+00:00
last_modified_at: 2026-11-12T11:40:18Z
tags: ["codex-cli", "gpt-6", "sol", "quota-management", "token-consumption", "billing", "cost-governance", "rate-limits"]
---

# The Token Spiral: Why GPT-6.1 Sol Is Eating Your Weekly Allowance


---

One conversation. 88.96 per cent of a weekly usage allowance. Gone in under an hour.

That is not a hypothetical. It is a verbatim report from GitHub issue #60530, filed on 10 November 2026, describing what happened when a Pro subscriber ran a medium-complexity refactoring session on GPT-6.1 Sol.[^1] The rate-limit community thread has now reached 1,044 replies and 4,649 views — the most-read active issue on the OpenAI developer forum this week.[^2]

This article explains what is happening, why there are two separate causes, and what you can do today to protect your allocation.

---

## Two Problems, One Symptom

The reports all describe the same experience: quota draining at a rate that feels disconnected from the work being done. The natural assumption is that this is a billing bug — and partly it is. But there is also a structural cause that would exist even if the billing were correct. Conflating them leads to waiting for a bug fix that will only partially solve the problem.

### Cause One: The Billing Regression

On approximately 5 November 2026, something changed in how GPT-6.1 Terra's cached input tokens are billed.[^3] The expected cached input rate — a 90 per cent reduction against the standard input rate — stopped applying. Tokens that should have been billed at the cached rate were instead being charged at the full input rate. For sessions with significant context overlap between turns, this can multiply the effective per-turn cost by a factor of three to ten.

This regression is distinct from the 40,000-account billing error reported on 20 October 2026, which involved incorrect account credit adjustments.[^4] The November regression is a pricing-tier misapplication: the right model, the wrong rate bracket. OpenAI has not publicly acknowledged the November regression as of this writing.

**Practical implication:** If your quota consumption increased sharply after 5 November without any change to your workflow, the billing regression is the most likely explanation. Watch the OpenAI status page and the community thread for a resolution timeline.

### Cause Two: The Turns-Squared Problem

The second cause predates the billing regression and will remain after it is fixed. It is a consequence of how agentic sessions accumulate context.

In a standard Codex CLI session, each turn carries:
- The full accumulated conversation history
- Tool call results appended to the context
- AGENTS.md and active skill instructions injected fresh each turn
- The JSON schema definitions of every registered MCP tool

Each of these grows with the session. The conversation history grows by the content of every assistant and tool response. Tool call results — particularly from commands like `git diff`, `cargo test`, or large file reads — can inject thousands of tokens per turn. By turn 20, a session that started at 5,000 tokens per turn may be sending 25,000 tokens per turn.

The relationship is approximately quadratic: a session of *n* turns carries roughly *n* squared cumulative token overhead, because each turn carries the cost of all prior turns. This is the token spiral. It is not a bug — it is an inherent property of the transformer attention mechanism applied to growing context. It was always true, but GPT-6.1 Sol's expanded capability means users run longer sessions than they did with Terra or Luna, pushing further into the quadratic regime.[^5]

A session that is "twice as long" is not twice as expensive. It is four times as expensive, or more, once MCP tool schemas and verbose tool outputs compound the per-turn baseline.

---

## Reading Your Own Numbers

Before switching models or restricting usage, establish whether your session costs are in the expected range. The `ccusage` tool provides per-session token breakdowns:

```bash
npx ccusage sessions --last 10
```

Look for two signals in the output:

1. **Per-turn token counts that jump mid-session** — indicates a large tool call result was appended to context. Identify the culprit command and suppress verbose output where possible.
2. **Session total that is disproportionate to session length** — indicates the quadratic effect is in play. Sessions over 30-40 turns are especially susceptible.

If your session costs spiked after 5 November and your per-turn counts look roughly consistent with prior weeks, the billing regression is the dominant factor. If per-turn costs are growing across the session, the turns-squared dynamic is contributing.

---

## Five Mitigations for Today

These are ordered roughly by effort, from immediate to structural.

### 1. Switch to Fast Mode

The `/mode` command in the Codex CLI TUI toggles between Ultra and Fast mode. Fast mode is approximately 2.5 times cheaper per turn.[^6] The trade-off is response latency and reasoning depth — Fast mode is not suitable for complex multi-step planning, but it handles most code-editing and file-search tasks adequately.

```text
/mode fast
```

For sessions where you are primarily making targeted edits rather than reasoning through architecture decisions, Fast mode is the most direct cost lever.

### 2. Route Subagents to Luna

GPT-6.1 Luna is the cheapest tier in the current model stack. For subagent tasks — code formatting, test generation, documentation updates, file operations — Luna's capability ceiling is sufficient and the cost is substantially lower:

```toml
# config.toml
[agents]
default_subagent_model = "gpt-6-luna"
```

This change keeps your primary agent on Sol while routing spawned subagents to Luna. In sessions that spawn multiple subagents, this can reduce total session cost by 40-60 per cent without affecting the quality of the primary agent's reasoning.

### 3. Prune Your MCP Tool Registry

Every registered MCP server injects its full tool schema into every turn. A GitHub MCP server exposing 93 tools adds roughly 55,000 tokens per turn before any work is done.[^7] If you are not using a server in a given session, disable it by setting `enabled = false` in `config.toml`:

```toml
# config.toml
[mcp_servers.github]
# ... existing config ...
enabled = false
```

Re-enable it only when you need it. The difference in per-turn baseline cost can be significant for long sessions.

### 4. Suppress Verbose Tool Output

Commands like `cargo test --verbose`, `git log --stat`, or `docker compose logs` produce output that is 60-90 per cent noise from the model's perspective. Use output-limiting flags:

```bash
# Instead of:   git log --stat
# Use:          git log --oneline -20

# Instead of:   cargo test --verbose
# Use:          cargo test 2>&1 | tail -50
```

You can also set output discipline globally in your AGENTS.md:

```markdown
## Output Discipline
When running shell commands, prefer concise output flags:
- git log: use --oneline and limit to 20 entries
- test runners: pipe output through tail -100
- package managers: use --quiet or --silent where available
```

### 5. Set a Session Budget Cap

The `rollout_budget` feature in `config.toml` sends a warning when a session approaches a token ceiling, giving you an explicit signal to wrap up:

```toml
[features.rollout_budget]
enabled = true
limit_tokens = 300000
reminder_at_remaining_tokens = [100000, 50000, 20000]
```

This does not prevent overrun, but it makes quota consumption visible in real time rather than a post-session surprise.

---

## What to Expect

The November billing regression is a defect and will be fixed. The turns-squared cost dynamic is structural and will not be. When the billing regression resolves, quota consumption will return to its pre-November baseline — which, for many users running GPT-6.1 Sol on longer agentic sessions, will still be higher than expected.

The mitigations above address both causes. Fast mode and Luna routing reduce per-turn cost regardless of whether the billing is correct. MCP tool pruning and output limiting reduce the context accumulation rate. The rollout budget makes the spiral visible before it consumes your allocation.

The community thread has demonstrated one consistent pattern across all the high-drain reports: the sessions that burned quota fastest were long, tool-heavy, and had large MCP registries active throughout. Those sessions were doing exactly what Codex CLI is designed to do — but doing it in a way that compounds both the structural cost curve and the billing regression. The five mitigations above break that compounding.

---

[^1]: GitHub issue #60530 — "One conversation used 89% of my weekly allowance on Pro." Filed 10 November 2026. OpenAI developer community.
[^2]: OpenAI Community Forum — rate-limit and usage quota thread. 1,044 replies, 4,649 views as of 10 November 2026. Related issues: #60425, #59690, #55876.
[^3]: Community analysis of GPT-6.1 Terra cached input billing regression, circa 5 November 2026. Identified by comparison of session-level token reports in the developer forum thread.
[^4]: OpenAI billing error — 40,000-account credit adjustment bug. First reported 20 October 2026. GitHub developer community.
[^5]: For a detailed breakdown of the MCP tax, context accumulation mechanics, and reasoning token overhead, see: *Diagnosing and Reducing Codex CLI Token Consumption: A Practitioner's Toolkit*. `2026-06-10-codex-cli-token-consumption-diagnosis-reduction-quota-drain-practitioner-toolkit.md`.
[^6]: Codex CLI TUI mode reference — Fast mode vs. Ultra mode token cost multipliers. Current documentation: `https://developers.openai.com/codex/models`.
[^7]: Measured MCP tool schema injection cost for a 93-tool GitHub MCP server: approximately 55,000 tokens per turn. Source: community benchmarking, OpenAI developer forum, May-June 2026.
