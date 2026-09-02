---
title: "Codex CLI Power-User Playbook: 15 Non-Obvious Features and Configuration Tricks for v0.137"
description: "A curated guide to lesser-known Codex CLI capabilities — from draft restoration on prompt cancellation to shell environment lockdown, parallel web search, and machine-readable diagnostics — targeting senior developers who already know the basics."
type: Technical Article
timestamp: 2026-06-04T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-06-04-codex-cli-power-user-playbook-non-obvious-features-configuration-tricks-v0137"
tags: ["codex-cli", "v0.137", "productivity", "power-user", "configuration", "tips", "advanced-workflows"]
date: 2026-06-04T09:00:00+00:00
last_modified_at: 2026-09-02T10:37:10+01:00
---
# Codex CLI Power-User Playbook: 15 Non-Obvious Features and Configuration Tricks for v0.137


Most Codex CLI guides cover the obvious: run `codex exec`, write an `AGENTS.md`, connect an MCP server. This article is not that guide. It collects fifteen non-obvious features — keyboard shortcuts, configuration keys, and behavioural quirks — that experienced users rely on daily but that rarely appear in introductory material. Everything here is current as of v0.137.0 stable (4 June 2026).[^1]

---

## 1 — Draft restoration on prompt cancellation

If you press `Ctrl+C` to cancel a prompt before visible output arrives, v0.137 now restores your draft text, any attached images, and your collaboration mode state.[^1] Previous versions silently discarded the draft. This means you can safely cancel a slow first-token response, tweak the prompt, and resubmit without retyping.

## 2 — `--ephemeral` for throwaway exec runs

When scripting one-shot tasks, pass `--ephemeral` to `codex exec` to skip writing session files to disk entirely.[^2]

```bash
codex exec --ephemeral "Summarise the last 10 git commits in this repo"
```

This keeps `~/.codex/sessions/` clean and avoids accumulating hundreds of stale transcript files in CI pipelines.[^2] Combine with `--ignore-user-config` and `--ignore-rules` for a fully isolated, reproducible run.

## 3 — `Ctrl+R` prompt history search

Inside the TUI, press `Ctrl+R` to search your previous prompt history.[^3] This works identically to Bash reverse search — type a substring and cycle through matches. Surprisingly few users know this exists, yet it eliminates the single biggest friction point: retyping complex prompts after a session restart.

## 4 — `shell_environment_policy` with `inherit = "none"`

By default, Codex strips variables containing `KEY`, `SECRET`, or `TOKEN` from the subprocess environment.[^4] But if you work with non-standard secret naming (e.g., `DATABASE_URL`, `STRIPE_SK`), that automatic filter misses them. Lock it down properly:

```toml
[shell_environment_policy]
inherit = "none"
set = { PATH = "/usr/local/bin:/usr/bin:/bin", HOME = "/home/dev" }
include_only = ["LANG", "TERM"]
```

Starting from an empty environment and adding only what you need is safer than starting from everything and trying to exclude the dangerous bits.[^4] The `"core"` preset (`PATH`, `HOME`, `TERM`) is a reasonable middle ground for most workflows.

## 5 — `model_reasoning_effort` gradients

The `model_reasoning_effort` key accepts five levels: `minimal`, `low`, `medium`, `high`, and `xhigh`.[^5] Most users set it once and forget it. Power users switch per task:

```bash
# Quick formatting task — minimal reasoning
codex exec -c model_reasoning_effort=low "Reformat this JSON file"

# Complex refactor — maximum reasoning
codex exec -c model_reasoning_effort=xhigh "Decompose this 800-line function into testable units"
```

The `-c` flag sets any TOML key inline without editing config files.[^5] At `xhigh`, expect roughly 3–5x the token cost but substantially better results on multi-step reasoning tasks.[^6]

## 6 — `--no-alt-screen` for scrollback preservation

By default, the TUI uses the terminal's alternate screen buffer, which means your session vanishes from scrollback when you exit.[^2] Pass `--no-alt-screen` (or set `tui.alternate_screen = "never"` in config) to keep everything in the main buffer:

```bash
codex --no-alt-screen
```

This is essential when you pipe terminal output to a log file, use `tmux capture-pane`, or simply want to scroll back through a session after exiting.[^7]

## 7 — `codex doctor --json` for machine-readable diagnostics

Everyone knows `codex doctor`. Fewer know `codex doctor --json`, which emits structured JSON suitable for automated health checks:[^2]

```bash
codex doctor --json | jq '.checks[] | select(.status == "fail")'
```

In CI, this lets you gate deployments on agent health. Combine with `--summary` for a compact overview or `--all` to expand truncated lists.[^2]

## 8 — `codex plugin list --json` for plugin automation

Similarly, `codex plugin list --json` (new in v0.137) outputs machine-readable plugin metadata with cached remote catalogue suggestions.[^1] This enables scripted plugin management across teams:

```bash
# Check which plugins are installed across all developer machines
codex plugin list --json | jq '[.[] | .name]'
```

## 9 — `.codexignore` for surgical context scoping

Place a `.codexignore` file at your repository root with gitignore-style patterns to exclude files from the agent's context window.[^3] This is more powerful than it sounds — excluding generated files, vendored dependencies, and large data directories can halve context consumption:

```gitignore
# .codexignore
vendor/
dist/
*.generated.ts
node_modules/
*.min.js
```

Every token saved on irrelevant context is a token available for reasoning about your actual code.[^8]

## 10 — `/compact` before pivoting tasks

When you finish one logical task and want to pivot to another within the same session, run `/compact` first.[^3] This summarises earlier turns into a condensed representation, freeing context window space for the new task. Without compaction, the agent carries forward irrelevant context from the previous task, increasing both cost and hallucination risk.

```mermaid
flowchart LR
    A[Task 1: Debug auth] --> B[/compact]
    B --> C[Task 2: Write tests]
    C --> D[/compact]
    D --> E[Task 3: Update docs]
```

## 11 — Profile stacking with `--profile`

Named profiles load from `$CODEX_HOME/<name>.config.toml` and layer on top of the base config.[^5] You can stack multiple profiles:

```bash
codex --profile security --profile verbose
```

This loads `~/.codex/security.config.toml` then `~/.codex/verbose.config.toml`, with later profiles overriding earlier ones. Practical profiles to maintain:

```toml
# ~/.codex/fast.config.toml
model = "gpt-5.3-codex"
model_reasoning_effort = "low"

# ~/.codex/careful.config.toml
model = "gpt-5.5"
model_reasoning_effort = "xhigh"
approval_policy = "on-request"
```

Switch context in seconds without editing any files.[^5]

## 12 — `--output-schema` for validated structured output

When you need Codex to produce data rather than prose, pass a JSON Schema file to `--output-schema`:[^2]

```bash
codex exec --output-schema schema.json \
  "Analyse this codebase and list all public API endpoints"
```

The agent's final message is validated against the schema before output. If validation fails, the run errors rather than silently producing malformed data. This is indispensable for pipelines that feed agent output into downstream systems.[^9]

## 13 — `codex sandbox` as a standalone isolation tool

The `codex sandbox` subcommand runs arbitrary shell commands under the same platform-native isolation (macOS Seatbelt, Linux Landlock/bubblewrap, Windows restricted tokens) that protects the agent loop — but without the agent.[^2] Use it to test untrusted scripts:

```bash
codex sandbox --permissions-profile read-only -- ./suspicious-script.sh
```

Add `--log-denials` on macOS to see exactly which system calls the sandbox blocks.[^10] This is invaluable for auditing third-party install scripts or CI actions before trusting them.

## 14 — Parallel web search in code mode

As of v0.137, standalone web searches execute in parallel within code-mode flows.[^1] This means that when the agent needs to look up multiple APIs or check several documentation pages, it issues concurrent search requests rather than sequential ones. Enable web search with `--search` or by setting `search = true` in your config:

```toml
search = true
```

The practical effect is faster research-heavy turns — the agent spends less wall-clock time waiting for search results.[^1]

## 15 — `project_root_markers` for non-Git projects

By default, Codex uses `.git` to detect the project root.[^5] If you work in monorepos with nested roots, or in projects that use Mercurial, Bazel, or other build systems as the canonical root marker, override this:

```toml
project_root_markers = [".git", "WORKSPACE", "BUILD.bazel", ".hg"]
```

Correct root detection affects which `AGENTS.md` file loads, where `.codex/config.toml` is found, and which directory the sandbox treats as the workspace root.[^5] Getting this wrong silently degrades every session.

---

## Putting it together

These fifteen features share a common thread: they reward intentional configuration. Codex CLI ships with sensible defaults, but the gap between a default installation and a properly configured one is the difference between a novelty and a production tool.

```mermaid
graph TD
    subgraph "Foundation"
        A[shell_environment_policy] --> B[.codexignore]
        B --> C[project_root_markers]
    end
    subgraph "Per-Task Tuning"
        D[--profile stacking] --> E[model_reasoning_effort]
        E --> F[--output-schema]
    end
    subgraph "Session Hygiene"
        G[/compact] --> H[--ephemeral]
        H --> I[--no-alt-screen]
    end
    subgraph "Automation"
        J[doctor --json] --> K[plugin list --json]
        K --> L[codex sandbox]
    end
```

Start with the foundation layer — environment lockdown, context scoping, and root detection. Add per-task tuning as you learn which tasks benefit from reasoning effort adjustments. Build session hygiene habits. Then automate health checks and plugin management across your team.

The official best practices documentation recommends treating Codex "less like a one-off assistant and more like a teammate you configure and improve over time".[^3] These fifteen features are how you do that configuring.

---

## Citations

[^1]: [Release 0.137.0 — openai/codex GitHub Releases](https://github.com/openai/codex/releases/tag/rust-v0.137.0), 4 June 2026.
[^2]: [Command line options — Codex CLI, OpenAI Developers](https://developers.openai.com/codex/cli/reference).
[^3]: [Best practices — Codex, OpenAI Developers](https://developers.openai.com/codex/learn/best-practices).
[^4]: [Advanced Configuration — Codex, OpenAI Developers](https://developers.openai.com/codex/config-advanced).
[^5]: [Configuration Reference — Codex, OpenAI Developers](https://developers.openai.com/codex/config-reference).
[^6]: [Prompt Caching 201 — OpenAI Cookbook](https://developers.openai.com/cookbook/examples/prompt_caching_201).
[^7]: [TUI Configuration — Codex CLI, OpenAI Developers](https://developers.openai.com/codex/cli/features).
[^8]: [Context Management Strategies for OpenAI Codex — Alex Merced](https://iceberglakehouse.com/posts/2026-03-context-openai-codex/).
[^9]: [Non-interactive mode — Codex, OpenAI Developers](https://developers.openai.com/codex/noninteractive).
[^10]: [Codex CLI `codex sandbox` Subcommand — Codex Knowledge Base](https://codex.danielvaughan.com/2026/06/04/codex-cli-sandbox-subcommand-standalone-command-isolation-platform-native-security/).
