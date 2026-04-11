---
title: "Legacy Model Retirement: Migration Guide for the April 14 Deadline"
date: 2026-04-11T17:00:00+00:00
tags:
  - models
  - migration
  - gpt-5.2
  - gpt-5.1
  - deadline
  - ci-cd
  - config-toml
---

![Sketchnote diagram for: Legacy Model Retirement: Migration Guide for the April 14 Deadline](/sketchnotes/articles/2026-04-11-legacy-model-retirement-april-14-migration-guide.png)

# Legacy Model Retirement: Migration Guide for the April 14 Deadline


---

On April 7, OpenAI removed six legacy models from the Codex model picker. On **April 14**, these models will be fully removed for ChatGPT sign-in users. If your workflows still reference any of these models — in `config.toml`, `AGENTS.md`, CI pipelines, or custom harnesses — they will break silently or fall back to defaults. This guide walks you through auditing and migrating before the deadline.

## What Is Being Removed

The following models are no longer available in the model picker (since April 7) and will be fully removed on April 14 for ChatGPT sign-in authentication:

| Retired Model | Original Role | Suggested Replacement |
|---|---|---|
| `gpt-5.2-codex` | Previous flagship | `gpt-5.4` or `gpt-5.3-codex` |
| `gpt-5.1-codex-mini` | Cost-efficient small model | `gpt-5.4-mini` |
| `gpt-5.1-codex-max` | Long-context heavy tasks | `gpt-5.4` |
| `gpt-5.1-codex` | Standard coding model | `gpt-5.3-codex` |
| `gpt-5.1` | General-purpose | `gpt-5.4` |
| `gpt-5` | Original GPT-5 | `gpt-5.4` |

## What Stays Available

| Model | Best For | Notes |
|---|---|---|
| `gpt-5.4` | New flagship — best quality | Default for most workflows |
| `gpt-5.4-mini` | Fast, cost-efficient tasks | Ideal for subagents, CI checks |
| `gpt-5.3-codex` | Code-specialised reasoning | Good balance of speed and quality |
| `gpt-5.2` | Still available via API key | Not in picker for ChatGPT sign-in |
| `gpt-5.3-codex-spark` | Ultra-fast (1000+ tok/s) | ChatGPT Pro only |

## Who Is Affected

- **ChatGPT sign-in users**: Full removal on April 14. Any `config.toml` or `AGENTS.md` referencing retired models will fail.
- **API key users**: Can still access older models via API key authentication. No immediate change, but these models will eventually be deprecated too.
- **CI/CD pipelines**: Any pipeline using `codex exec` with a hardcoded model name will break if it authenticates via ChatGPT sign-in.

## Step-by-Step Migration

### 1. Audit Your Configurations

Search every place a model name might be hardcoded:

```bash
# config.toml
grep -r "gpt-5\.\(1\|2\)-codex\|gpt-5\.1\b\|gpt-5\b" ~/.codex/config.toml

# AGENTS.md files in all repos
find ~/projects -name "AGENTS.md" -exec grep -l "gpt-5\.\(1\|2\)-codex\|gpt-5\.1\b\|gpt-5\b" {} \;

# CI pipeline files
grep -rn "gpt-5\.\(1\|2\)-codex\|gpt-5\.1\b\|gpt-5\b" .github/workflows/ .gitlab-ci.yml Jenkinsfile 2>/dev/null

# Subagent TOML configs
find . -name "*.toml" -exec grep -l "gpt-5\.\(1\|2\)-codex" {} \;
```

### 2. Update config.toml

Replace the model in your global config:

```toml
# Before
[model]
name = "gpt-5.2-codex"

# After — pick based on your needs
[model]
name = "gpt-5.4"          # best quality
# name = "gpt-5.4-mini"   # fastest, cheapest
# name = "gpt-5.3-codex"  # code-specialised balance
```

### 3. Update AGENTS.md Model Hints

If your `AGENTS.md` files specify model preferences:

```markdown
<!-- Before -->
Use gpt-5.2-codex for all code generation tasks.

<!-- After -->
Use gpt-5.4 for code generation. Use gpt-5.4-mini for routine checks and subagent tasks.
```

### 4. Update CI Pipelines

For `codex exec` in CI:

```yaml
# Before
- run: codex exec --model gpt-5.1-codex "Fix the failing tests"

# After
- run: codex exec --model gpt-5.4-mini "Fix the failing tests"
```

Consider using `gpt-5.4-mini` in CI for cost efficiency — it handles routine tasks well at lower token cost.

### 5. Update Subagent Configurations

If you use TOML-based subagent orchestration:

```toml
# Before
[[agents]]
name = "reviewer"
model = "gpt-5.2-codex"

# After
[[agents]]
name = "reviewer"
model = "gpt-5.4"
```

### 6. Test Before the Deadline

Run a dry test with your updated configs:

```bash
# Quick smoke test
codex --model gpt-5.4 "echo hello world"

# Verify subagents work
codex exec --model gpt-5.4-mini "List the files in the current directory"
```

## Migration Strategy for Teams

| Scenario | Recommended Model | Rationale |
|---|---|---|
| Primary development | `gpt-5.4` | Best reasoning, highest quality |
| Subagents / parallel tasks | `gpt-5.4-mini` | Fast, cheap, good enough for scoped tasks |
| Code review / adversarial review | `gpt-5.3-codex` | Code-specialised, strong at finding bugs |
| CI/CD automation | `gpt-5.4-mini` | Cost-efficient for repetitive pipeline tasks |
| Long-running agentic workflows | `gpt-5.4` | Better context handling for complex multi-step tasks |

## API Key Workaround

If you need more time, switch to API key authentication temporarily:

```bash
export OPENAI_API_KEY="sk-..."
codex --model gpt-5.2-codex "your prompt"
```

This bypasses the ChatGPT sign-in model restrictions. Note: this may incur different pricing and the older models will eventually be deprecated for API users too.

## Key Dates

- **April 7**: Models removed from picker UI
- **April 14**: Models fully removed for ChatGPT sign-in
- **TBD**: API key deprecation (expected Q3 2026 based on historical patterns)

## Further Reading

- [Codex CLI Model Lifecycle article](/2026/04/07/codex-cli-model-lifecycle-deprecations-migrations/) — full timeline and deprecation history
- [Official Codex Changelog](https://developers.openai.com/codex/changelog)
- [OpenAI Deprecations page](https://platform.openai.com/docs/deprecations)

---

*Written 2026-04-11. Sources: [developers.openai.com/codex/changelog](https://developers.openai.com/codex/changelog), [platform.openai.com/docs/deprecations](https://platform.openai.com/docs/deprecations), [releasebot.io/updates/openai/codex](https://releasebot.io/updates/openai/codex)*
