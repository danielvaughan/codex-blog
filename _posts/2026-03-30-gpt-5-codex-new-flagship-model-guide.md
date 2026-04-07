---
title: "gpt-5-codex: The New Codex Flagship and What It Means for Your Workflow"
layout: default
parent: "Articles"
nav_order: 115
---

# gpt-5-codex: The New Codex Flagship and What It Means for Your Workflow

**Date:** 2026-03-30
**Tags:** gpt-5-codex, model-update, model-selection, gpt-5-codex-mini, chatgpt-login, model-consolidation

In late March 2026, OpenAI shipped two new models — `gpt-5-codex` and `gpt-5-codex-mini` — announced by @thsottiaux. The naming is deliberate: these are not "gpt-5.4 Codex" variants but a distinct model tier with a simplified name, representing a consolidation of the Codex model line.

This article explains what changed, what the new models are best at, and how to migrate your configuration.

---

## What Changed

Until this launch, the Codex model lineup was a numbered sequence: gpt-5.1-codex-max → gpt-5.2-codex → gpt-5.3-codex, plus the general-purpose gpt-5.4 family available across Codex surfaces.

The arrival of `gpt-5-codex` introduces a cleaner naming convention that drops the sub-version number. @thsottiaux's announcement framed it directly:

> *"gpt-5-codex — fast for small things, working hard when it matters — solid jump in produced code quality"*

The key signals:
- **"Fast for small things"** — smarter routing of effort. Small fixes and lookups get fast responses without burning reasoning budget
- **"Working hard when it matters"** — automatic effort calibration for complex tasks (no manual `reasoning_effort` tuning required for most workflows)
- **"Solid jump in code quality"** — the phrasing @thsottiaux has not used for any previous model update

Meanwhile, `gpt-5-codex-mini` ships with a compelling economics story for Plus, Edu, and Team plan users: **up to 4x more included usage** than GPT-5.4-mini, plus a 50% increase in overall usage limits for those plans. Pro users get priority processing.

---

## The Model Consolidation Context

Simon Willison observed on March 5 that GPT-5.4 "beats coding specialist GPT-5.3-Codex on all relevant benchmarks", and wondered: *"I wonder if we'll get a 5.4 Codex or if that model line has now been merged into main?"*[^1]

The answer from late March is: neither exactly. Instead of a "gpt-5.4-codex", OpenAI shipped `gpt-5-codex` — a model name that drops version numbers entirely. This is consistent with the platform trajectory: as the frontier general-purpose model absorbs coding capabilities previously requiring specialist fine-tuning, the naming can simplify. The Codex product line retains its branding while the model underneath becomes increasingly frontier-general.

**What this means practically:**
- If you were on `gpt-5.3-codex` for coding tasks, migrate to `gpt-5-codex`
- If you were on `gpt-5.4` for Codex work, also migrate to `gpt-5-codex` — it's the better choice for Codex-specific workflows
- The `-mini` variant replaces `gpt-5.4-mini` as the subagent workhorse, with significantly better economics for most plans

---

## Model Comparison

| Model | Best for | Notes |
|-------|---------|-------|
| `gpt-5-codex` | Main tasks, complex code changes, architecture work | 🆕 New flagship |
| `gpt-5-codex-mini` | Exploration, large-file review, subagent delegation | 🆕 4x usage (Plus/Edu/Team) |
| `gpt-5.4` | Non-Codex surfaces (API, ChatGPT), computer use workflows | Still valid; gpt-5-codex is preferred within Codex |
| `gpt-5.4-mini` | Superseded by gpt-5-codex-mini for most uses | Remains available |
| `gpt-5.3-codex` | Previous flagship | Keep for reproducibility; not recommended for new work |
| `gpt-5.3-codex-spark` | Real-time coding (Pro only, 128k, text-only) | Cerebras hardware; distinct use case |

---

## How to Use the New Models

### Interactive CLI

```bash
codex -m gpt-5-codex
codex -m gpt-5-codex-mini
```

### config.toml

```toml
model = "gpt-5-codex"

# For agents that do exploration/review work:
[agents.explorer]
model = "gpt-5-codex-mini"
description = "Large-file search and codebase exploration"
```

### Subagent delegation pattern

```toml
# config.toml — tiered topology with the new models
model = "gpt-5-codex"

[agents.reviewer]
model = "gpt-5-codex-mini"
description = "Review output from implementer agents"

[agents.implementer]
model = "gpt-5-codex"
description = "Complex implementation tasks"

[agents.explorer]
model = "gpt-5-codex-mini"
description = "Codebase search, large-file review"
```

Use `gpt-5-codex-mini` where you'd previously use `gpt-5.4-mini` — the economics are significantly better on Plus/Edu/Team plans.

---

## ChatGPT Login (No API Key Needed)

Alongside the model launch, OpenAI confirmed ChatGPT Plus/Pro login now works natively in the Codex CLI — no separate API key setup required:

```bash
brew install codex
codex        # opens onboarding, lets you log in with ChatGPT credentials
```

This makes `gpt-5-codex` accessible to anyone on the ChatGPT Plus or Pro plan without touching the API dashboard. For Enterprise users, existing SCIM/SSO flows remain unchanged.

---

## API Availability

`gpt-5-codex` is available in the **Responses API** directly (not the legacy `chat/completions` endpoint, which was removed in February 2026):

```python
# Using the OpenAI Responses API
from openai import OpenAI

client = OpenAI()
response = client.responses.create(
    model="gpt-5-codex",
    input="Review this diff and identify any security issues..."
)
```

In `config.toml`, ensure you're on the Responses API:

```toml
wire_api = "responses"
model = "gpt-5-codex"
```

**Note:** `gpt-5-codex` is **not available in ChatGPT** (it's a Codex-surface and Responses API model). For ChatGPT-based workflows, `gpt-5.4` remains the appropriate choice.

---

## Migration Checklist

If you're moving from earlier models:

- [ ] Update `model` in `~/.codex/config.toml` to `gpt-5-codex`
- [ ] Update subagent configs: replace `gpt-5.4-mini` with `gpt-5-codex-mini` in `.codex/agents/*.toml`
- [ ] Update any hardcoded model names in `codex exec` scripts
- [ ] If on Plus/Edu/Team: verify usage limits increased 50% in account settings
- [ ] For CI/CD pipelines: update `--model` flags in GitHub Actions / CI scripts

---

## Related Notes

- [Model lineup overview](../notes/models-and-features.md) — full benchmark data and version history
- [GPT-5.4 mini subagent delegation](./2026-03-30-gpt54-mini-codex-subagent-delegation.md) — tiered inference architecture (still relevant; mini variant now replaced by gpt-5-codex-mini)
- [Model selection guide](./2026-03-26-codex-cli-model-selection.md) — decision framework (update model names as above)

---

[^1]: Simon Willison, "Introducing GPT-5.4" (March 5, 2026) — https://simonwillison.net/2026/Mar/5/introducing-gpt54/
