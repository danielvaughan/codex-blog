---
title: "Reasoning Effort Tuning: Minimal to xhigh for Cost and Speed"
date: 2026-03-27
summary: "How to tune model_reasoning_effort and plan_mode_reasoning_effort for every workflow. The five levels explained with real cost/speed tradeoffs and subagent economics."
tags:
  - models
  - model-selection
  - cost-optimization
  - config-toml
  - codex-cli
  - performance
  - cost
  - configuration
---
![Sketchnote diagram for: Reasoning Effort Tuning: Minimal to xhigh for Cost and Speed](/sketchnotes/articles/2026-03-27-reasoning-effort-tuning.png)

# Reasoning Effort Tuning: Minimal to xhigh for Cost and Speed

Codex CLI's reasoning engine has a single knob that dramatically affects cost, speed, and quality: `model_reasoning_effort`. Most developers leave it at the default (`medium`) and don't think about it again. That's the right call for interactive work — but it leaves significant cost savings and quality gains on the table for everything else.

---

## The Five Effort Levels

| Level | Tokens used | Speed | Best for |
|-------|-------------|-------|----------|
| `minimal` | Fewest | Fastest | Trivial tasks: rename, format, single-line fix |
| `low` | Low | Fast | Boilerplate generation, well-defined data transforms |
| `medium` ✅ | Moderate | Responsive | **Default. Everyday interactive coding** |
| `high` | High | Slower | Multi-file refactors, complex debugging, architectural decisions |
| `xhigh` | Most | Slowest | Long-horizon autonomous tasks, hard algorithmic problems, deep security audits |

**Default:** `medium` — OpenAI's recommended all-day driver. Balances quality and cost for most interactive sessions.

**Newest addition:** `xhigh` — introduced for non-latency-sensitive tasks where quality matters more than speed. Codex "thinks for an even longer period" before responding.

---

## How to Set It

### In `~/.codex/config.toml` (global default)

```toml
model = "gpt-5.4"
model_reasoning_effort = "medium"          # applies to all tasks
plan_mode_reasoning_effort = "high"        # override for /plan mode only
```

### Via the `-e` flag (per-session)

```bash
codex -e high "Refactor the auth module to use JWT"
codex -e minimal "Add a docstring to this function"
codex -e xhigh "Find the race condition in this event loop"
```

### In profiles (per-workflow)

Profiles let you pre-configure effort per context. Add to `config.toml`:

```toml
[profiles.daily]
model_reasoning_effort = "medium"

[profiles.deep-work]
model_reasoning_effort = "high"
model = "gpt-5.4"

[profiles.triage]
model_reasoning_effort = "low"
model = "gpt-5.4-mini"
```

Launch with: `codex --profile deep-work "..."` or set `default_profile = "daily"`.

### Slash command (mid-session)

Type `/effort high` in the Codex TUI to switch reasoning effort without restarting the session.

---

## Plan Mode Override

`plan_mode_reasoning_effort` is a separate config key that kicks in when you use `/plan`. Since planning is where quality matters most (a bad plan compounds into a bad implementation), it makes sense to run plan mode at a higher effort than your default:

```toml
model_reasoning_effort = "medium"    # interactive coding
plan_mode_reasoning_effort = "high"  # planning always gets more thought
```

When `plan_mode_reasoning_effort` is unset, plan mode uses its built-in preset default (currently `high`).

---

## Token Economics: What You're Actually Paying For

Reasoning models use **reasoning tokens** in addition to input/output tokens. Reasoning tokens are "thinking work" — the model uses them internally to break down problems. They count toward your usage but don't appear in the conversation.

The multiplier is steep at `xhigh`. Budget accordingly:

| Effort | Approximate reasoning token multiplier (vs `medium`) |
|--------|------------------------------------------------------|
| `minimal` | ~0.1x |
| `low` | ~0.3x |
| `medium` | 1x (baseline) |
| `high` | ~3–5x |
| `xhigh` | ~8–15x |

*Exact multipliers are model-dependent and not publicly documented. These are approximate based on community benchmarks.*

### GPT-5.1-Codex-Max efficiency note

GPT-5.1-Codex-Max achieves better SWE-bench Verified performance than GPT-5.1-Codex **at the same `medium` effort** while using **30% fewer thinking tokens**. If you're on the Max tier, you get better quality at lower cost — no effort tuning required.

---

## Subagent Economics: The Real Win

The effort tuning story is most powerful in multi-subagent workflows. When you spawn subagents via TOML, each runs independently and incurs its own reasoning cost. The pattern: **parent agent at `high`/`medium` for orchestration; subagents at `low`/`minimal` for execution**.

Example subagent config in `codex-tasks.toml`:

```toml
[[agents]]
id = "planner"
prompt = "Break this feature into 5 non-overlapping implementation tasks"
reasoning_effort = "high"      # plan quality matters

[[agents]]
id = "implementer-1"
prompt = "Implement task 1: add the database migration"
reasoning_effort = "low"       # task is well-defined, save tokens

[[agents]]
id = "tester"
prompt = "Write unit tests for the migration"
reasoning_effort = "low"       # mechanical work

[[agents]]
id = "reviewer"
prompt = "Review all changes for security issues"
reasoning_effort = "high"      # security needs careful reasoning
```

This pattern can reduce overall token spend by 50–70% vs running everything at `medium`, while maintaining quality where it matters.

---

## Decision Framework

```
Is the task well-defined, single-file, or mechanical?
  → minimal or low

Is it a standard interactive session or multi-file edit?
  → medium (default, leave it)

Does it involve planning, architecture, or debugging a hard problem?
  → high (or set plan_mode_reasoning_effort = "high")

Is it a long-running autonomous task, security audit, or algorithmic hard problem?
  → xhigh (but set expectations: this is slow and expensive)

Is it a subagent executing a clearly-specified task?
  → low or minimal (your orchestrator already did the hard thinking)
```

---

## Common Mistakes

- ❌ **Running everything at `high`** — 3–5x the token cost for routine tasks. Use profiles to scope it.
- ❌ **Leaving plan mode at `medium`** — plan quality is where you get compounding returns. Always run plan mode at `high` or better.
- ❌ **Treating `xhigh` as a default** — it's a specialist tool for hard problems, not a quality upgrade for everything.
- ❌ **Not using profiles** — the `-e` flag is for one-offs; profiles are for habits.
- ❌ **Same effort for all subagents** — orchestrators need `high`, executors need `low`. Match effort to role.

---

## Config Reference

```toml
# Full effort configuration in config.toml
model_reasoning_effort = "medium"          # global default
plan_mode_reasoning_effort = "high"        # /plan mode override

[profiles.ci]
model_reasoning_effort = "low"             # fast CI runs
model = "gpt-5.4-mini"

[profiles.audit]
model_reasoning_effort = "xhigh"           # security/compliance reviews
model = "gpt-5.4"
```

Valid values: `"minimal"` | `"low"` | `"medium"` | `"high"` | `"xhigh"` (model-dependent).

---

*Published 2026-03-27. Based on official Codex CLI config reference, changelog, and community benchmarks.*
