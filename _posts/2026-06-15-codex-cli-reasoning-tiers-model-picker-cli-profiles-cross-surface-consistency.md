---
title: "Codex CLI Reasoning Tiers: Mapping the June 2026 Model Picker to CLI Profiles for Cross-Surface Consistency"
type: Technical Article
timestamp: 2026-06-15T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-06-15-codex-cli-reasoning-tiers-model-picker-cli-profiles-cross-surface-consistency"
tags: ["codex-cli", "reasoning-effort", "model-picker", "named-profiles", "cross-surface", "config-toml", "gpt-5.5", "cost-optimisation", "enterprise"]
date: 2026-06-15T09:00:00+00:00
last_modified_at: 2026-09-04T00:13:32+01:00
---
# Codex CLI Reasoning Tiers: Mapping the June 2026 Model Picker to CLI Profiles for Cross-Surface Consistency


---

On 10 June 2026, OpenAI replaced the ChatGPT model picker with a simplified six-tier reasoning menu: Instant, Medium, High, Extra High, Pro Standard, and Pro Extended[^1]. The labels are friendlier, but they hide a problem every team running both the Codex App and Codex CLI will eventually hit: a developer selects "High" in the App, then switches to the CLI and gets a different model at a different reasoning effort, producing inconsistent behaviour for the same task. This article maps the new tiers to concrete `config.toml` profiles, explains the token economics behind each level, and provides a team standardisation playbook that keeps agent behaviour consistent regardless of which surface a developer uses.

## What Changed on 10 June

Before the update, ChatGPT exposed model names directly — GPT-5.5, GPT-5.5 Thinking, and so on — alongside a separate thinking-level toggle (Standard, Extended, Heavy). The June 10 redesign collapsed both dimensions into a single selector[^1]:

| Picker Label | Previous Name | Underlying Model | Reasoning Effort | Access |
|---|---|---|---|---|
| Instant | GPT-5.5 Instant | GPT-5.5 | `none` / `minimal` | Plus, Pro, Enterprise, Edu |
| Medium | Thinking Standard | GPT-5.5 | `medium` | Plus, Pro, Enterprise, Edu |
| High | Thinking Extended | GPT-5.5 | `high` | Plus, Pro, Enterprise, Edu |
| Extra High | Thinking Heavy | GPT-5.5 | `xhigh` | Pro only |
| Pro Standard | Pro Standard | GPT-5.5 | `high` + extended compute | Pro only |
| Pro Extended | Pro Extended | GPT-5.5 | `xhigh` + extended compute | Pro only |

The model underneath is the same across all six tiers — GPT-5.5[^2]. What changes is the reasoning budget the model allocates before producing output. Instant skips extended reasoning entirely; Extra High and the Pro tiers let the model think for significantly longer, burning substantially more tokens in the process[^3].

## Why This Matters for CLI Users

The Codex App now abstracts away `model_reasoning_effort` behind friendly labels. The CLI does not. A developer who picks "High" in the App and then runs `codex` from a terminal with the default `config.toml` will land on `medium` reasoning effort — the CLI's default for GPT-5.5[^4]. The agent will produce noticeably different results for complex tasks: shallower analysis, fewer edge cases considered, and occasionally missed architectural concerns that the App session handled correctly.

For teams using both surfaces — and the June 2026 data suggests most enterprise teams do[^5] — this divergence creates a subtle quality inconsistency that is difficult to debug because neither surface reports the other's settings.

## Mapping Tiers to CLI Profiles

The solution is named profiles in `config.toml` that mirror the App's tier labels exactly. Since Codex CLI moved to per-file profile configuration in v0.135[^6], each profile lives in its own file under `~/.codex/`:

### Instant Profile

```toml
# ~/.codex/instant.config.toml
model = "gpt-5.5"
model_reasoning_effort = "none"
model_reasoning_summary = "none"
model_verbosity = "low"
```

Best for: quick lookups, simple renames, trivial formatting fixes. Latency is minimal and token cost is lowest[^3].

### Medium Profile

```toml
# ~/.codex/medium.config.toml
model = "gpt-5.5"
model_reasoning_effort = "medium"
```

This is the default reasoning tier and the recommended starting point for most development work. GPT-5.5 defaults to `medium` reasoning effort when no override is specified[^4], so this profile mainly serves as an explicit anchor — ensuring the developer knows they are matching the App's "Medium" tier rather than relying on an implicit default.

### High Profile

```toml
# ~/.codex/high.config.toml
model = "gpt-5.5"
model_reasoning_effort = "high"
```

Best for: code review, refactoring decisions, security analysis, and any task where missing an edge case costs more than the extra reasoning tokens[^7].

### Extra High Profile

```toml
# ~/.codex/xhigh.config.toml
model = "gpt-5.5"
model_reasoning_effort = "xhigh"
```

Reserve this for genuinely hard problems — architectural decisions, complex migration planning, and security audits. The token burn rate roughly doubles compared to `high`[^3]. This tier maps to the App's "Extra High" and is only available to Pro subscribers in the App, though CLI users with API keys can access `xhigh` regardless of their ChatGPT subscription tier[^8].

## Using Profiles

Launch a session with a specific profile:

```bash
codex --profile high
```

Or set a default in your base `config.toml`:

```toml
# ~/.codex/config.toml
model = "gpt-5.5"
model_reasoning_effort = "medium"
```

The `/effort` slash command allows mid-session switching without restarting[^9]:

```
/effort high
```

## The Token Economics of Each Tier

Understanding the cost implications is essential for budget-conscious teams. GPT-5.5 pricing sits at \$5.00 per million input tokens and \$30.00 per million output tokens[^10]. Reasoning tokens — the hidden "thinking" tokens the model generates internally — count as output tokens but are not visible in the response[^3].

```mermaid
graph LR
    A[Instant<br/>none/minimal] -->|~1x base cost| B[Medium<br/>medium effort]
    B -->|~1.5-2x| C[High<br/>high effort]
    C -->|~2-3x| D[Extra High<br/>xhigh effort]
    D -->|~3-5x| E[Pro Extended<br/>xhigh + extended]

    style A fill:#2d6a4f,color:#fff
    style B fill:#40916c,color:#fff
    style C fill:#e9c46a,color:#000
    style D fill:#e76f51,color:#fff
    style E fill:#9b2226,color:#fff
```

The multipliers above are approximate and task-dependent. A simple code formatting task may see negligible difference between Instant and High. A complex architectural review can see reasoning tokens exceed output tokens by 5:1 at `xhigh`[^3].

### Cached Input Discount

GPT-5.5 cached input tokens cost \$0.50 per million — a 90% discount on standard input pricing[^10]. For teams running repeated sessions against the same codebase, this makes `high` and `xhigh` significantly more affordable on second and subsequent turns, since AGENTS.md files, system prompts, and tool definitions all hit the cache.

## Plan Mode Reasoning Effort

Codex CLI supports a separate `plan_mode_reasoning_effort` key that activates when you use the `/plan` command[^9]. This allows a useful pattern: run interactive sessions at `medium` but escalate to `high` when the agent enters planning mode:

```toml
# ~/.codex/config.toml
model = "gpt-5.5"
model_reasoning_effort = "medium"
plan_mode_reasoning_effort = "high"
```

This mirrors a common human workflow — think harder when planning, move faster when executing — and keeps costs manageable while ensuring plans are thorough.

## Cross-Surface Consistency for Teams

### The requirements.toml Approach

Enterprise administrators can enforce a minimum reasoning floor across the organisation using `requirements.toml`[^11]. While `requirements.toml` cannot directly set reasoning effort, it can restrict available models and enforce managed configuration bundles — a feature that landed in v0.137 and was extended with cloud-managed config bundles in June 2026[^12]:

```toml
# requirements.toml (managed by admin)
[model]
allowed = ["gpt-5.5"]
```

### Repository-Level AGENTS.md

For project-specific consistency, include reasoning guidance in the repository's `AGENTS.md`:

```markdown
## Reasoning Guidelines

- Code review tasks: use `high` reasoning effort
- Formatting and linting: `none` or `minimal` is sufficient
- Security-sensitive changes: always use `xhigh`
- Default for general development: `medium`
```

This does not enforce profile selection, but it primes the agent to behave consistently and gives team members a shared reference for which tier to select.

### Shared Profile Distribution via Plugins

Teams can distribute standardised profiles through the Codex plugin system[^13]. A plugin manifest can bundle profile configuration files alongside skills and hooks, ensuring every team member has identical tier definitions:

```json
{
  "name": "team-profiles",
  "version": "1.0.0",
  "skills": [],
  "config_files": [
    "instant.config.toml",
    "medium.config.toml",
    "high.config.toml",
    "xhigh.config.toml"
  ]
}
```

## The "Show Additional Models" Escape Hatch

The June 10 update also introduced a "Show additional models" toggle in the App's web settings, revealing legacy options including o3, o4-mini, and 4.1[^1]. For CLI users, these models remain accessible via `config.toml` without any toggle — but most are now retired from the consumer ChatGPT picker and approaching API deprecation windows[^14]. Teams standardising on the new tier model should treat these legacy options as transitional and plan their removal from CI/CD profiles before the deprecation deadlines.

## A Practical Team Standardisation Checklist

1. **Audit current usage**: Run `codex doctor --json` and check what model and reasoning effort each team member is actually using[^15].

2. **Create shared profiles**: Distribute four standard profile files (instant, medium, high, xhigh) through your plugin marketplace or repository.

3. **Set sensible defaults**: Configure the base `config.toml` with `medium` as the default and `high` for plan mode.

4. **Document tier selection**: Add reasoning guidelines to your project's `AGENTS.md` so developers know which tier to use for which task.

5. **Monitor token consumption**: Use the workspace analytics dashboard (available for Business and Enterprise accounts) to track whether `xhigh` usage is justified by measurably better outcomes[^16].

6. **Align App and CLI**: Communicate to the team that the App's "High" equals `--profile high` in the CLI. Post the mapping table in your team wiki.

## Limitations and Caveats

- **Pro Standard and Pro Extended** tiers use extended compute budgets that are not directly replicable through the CLI's `model_reasoning_effort` key alone. These tiers involve server-side resource allocation tied to Pro subscriptions[^1]. CLI users with API keys can use `xhigh` but will not get the exact same extended compute behaviour. ⚠️

- **Automatic Instant-to-Medium switching**: The App offers a setting where Instant auto-switches to Medium for harder questions[^2]. The CLI has no equivalent — you get exactly the effort level you configure. For teams that rely on this adaptive behaviour, the CLI's explicit profile model is actually an advantage: deterministic reasoning depth means reproducible results.

- **Token visibility**: Reasoning tokens are not surfaced in the CLI's standard output. Use `codex doctor` or the workspace analytics dashboard to understand actual token consumption per session[^15].

## Conclusion

The simplified model picker is a usability improvement for casual ChatGPT users, but it creates a translation gap for teams working across the Codex App and CLI. Named profiles that map one-to-one with the picker's tiers close that gap. The investment is small — four TOML files and a team convention — but the payoff is significant: consistent agent behaviour, predictable costs, and no more debugging sessions that turn out to be reasoning-effort mismatches in disguise.

---

## Citations

[^1]: OpenAI, "ChatGPT Release Notes — June 10, 2026: Simplified model picker", [https://help.openai.com/en/articles/6825453-chatgpt-release-notes](https://help.openai.com/en/articles/6825453-chatgpt-release-notes)

[^2]: OpenAI, "GPT-5.5 in ChatGPT", [https://help.openai.com/en/articles/11909943](https://help.openai.com/en/articles/11909943)

[^3]: OpenAI, "Reasoning models — reasoning effort and token economics", [https://developers.openai.com/api/docs/guides/reasoning](https://developers.openai.com/api/docs/guides/reasoning)

[^4]: OpenAI, "Using GPT-5.5 — default reasoning effort", [https://developers.openai.com/api/docs/guides/latest-model](https://developers.openai.com/api/docs/guides/latest-model)

[^5]: OpenAI, "Codex is now generally available — 4 million weekly developers", [https://openai.com/index/codex-now-generally-available/](https://openai.com/index/codex-now-generally-available/)

[^6]: OpenAI, "Codex CLI Advanced Configuration — profile files", [https://developers.openai.com/codex/config-advanced](https://developers.openai.com/codex/config-advanced)

[^7]: Arsturn, "GPT-5 Reasoning Effort Levels Explained", [https://www.arsturn.com/blog/gpt-5-reasoning-effort-levels-explained](https://www.arsturn.com/blog/gpt-5-reasoning-effort-levels-explained)

[^8]: OpenAI, "OpenAI API Pricing — GPT-5.5", [https://openai.com/api/pricing/](https://openai.com/api/pricing/)

[^9]: Codex Knowledge Base, "Reasoning Effort Tuning: Minimal to xhigh for Cost and Speed", [https://codex.danielvaughan.com/2026/03/27/reasoning-effort-tuning/](https://codex.danielvaughan.com/2026/03/27/reasoning-effort-tuning/)

[^10]: OpenAI, "API Pricing — GPT-5.5: \$5/\$30 per 1M tokens", [https://openai.com/api/pricing/](https://openai.com/api/pricing/)

[^11]: OpenAI, "Codex CLI — requirements.toml policy enforcement", [https://developers.openai.com/codex/cli](https://developers.openai.com/codex/cli)

[^12]: OpenAI, "Codex Changelog — v0.137: cloud-managed config bundles", [https://developers.openai.com/codex/changelog](https://developers.openai.com/codex/changelog)

[^13]: OpenAI, "Codex Plugin System — bundling skills and configuration", [https://deepwiki.com/openai/codex/5.11-plugins-system](https://deepwiki.com/openai/codex/5.11-plugins-system)

[^14]: OpenAI, "Model deprecations and retirements", [https://help.openai.com/en/articles/9624314-model-release-notes](https://help.openai.com/en/articles/9624314-model-release-notes)

[^15]: OpenAI, "Codex CLI — codex doctor diagnostics", [https://developers.openai.com/codex/cli/features](https://developers.openai.com/codex/cli/features)

[^16]: OpenAI, "Codex Pricing — workspace analytics", [https://developers.openai.com/codex/pricing](https://developers.openai.com/codex/pricing)
