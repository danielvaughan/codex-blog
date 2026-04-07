---
title: "codex-plugin-cc: OpenAI Ships Codex Inside Claude Code"
date: 2026-03-31
tags: [codex-cli, claude-code, plugin, cross-model, strategy]
---

On March 31, 2026, OpenAI published [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc) — an official plugin that lets Claude Code users run Codex reviews and delegate tasks without leaving their session. Within hours it had 3,700+ GitHub stars. This is OpenAI shipping its agent *inside a competitor's tool*.

## What It Does

Six slash commands, all runnable in the background:

| Command | Purpose |
|---------|---------|
| `/codex:review` | Read-only code review on uncommitted changes (supports `--base <ref>`) |
| `/codex:adversarial-review` | Steerable challenge review — questions design decisions and pressure-tests assumptions |
| `/codex:rescue` | Delegates investigation/fixes to Codex via a subagent |
| `/codex:status` | Shows running and completed Codex jobs |
| `/codex:result` | Retrieves output; includes session IDs for `codex resume` |
| `/codex:cancel` | Terminates active background operations |

All commands support `--background` and `--wait` flags. The plugin wraps the local Codex CLI binary — no separate runtime, same credentials, same usage limits.

## Installation

```bash
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

Requirements: Node.js 18.18+, ChatGPT subscription or OpenAI API key.

## Why This Matters Strategically

1. **Ecosystem penetration over platform lock-in.** Claude Code accounts for ~4% of public GitHub commits (~135K/day) and ~$2.5B annualised revenue. OpenAI cannot ignore that install base — so they're embedding Codex *inside* it rather than competing for switching.

2. **Cross-model adversarial review becomes first-class.** The `/codex:adversarial-review` command formalises what power users already do manually: have one model critique another's work. Claude Code authors code; Codex acts as sceptical reviewer. This is the strongest use case — one developer noted: *"Claude more often finds big-picture or taste issues with Codex. And Codex more often finds correctness and code quality issues with Claude."*

3. **The "review gate" pattern.** An optional feature blocks Claude Code from finalising changes until Codex has reviewed them. This is essentially automated cross-model CI — with cost implications (feedback loops between agents can spike usage).

4. **Signal for the market.** If Anthropic or Google reciprocate with reverse plugins, competition shifts from raw model performance to integrated ecosystem maturity. The era of "pick one tool" may give way to "pick a primary agent, supplement with plugins."

## Relevance to Daniel's Work

- **Agentic pod architecture:** This plugin is a concrete implementation of the cross-model review pattern Daniel writes about. The adversarial review is exactly the "second opinion" layer in his pod design.
- **Dual-tool workflow validation:** Daniel already uses Codex for throughput/precision and Claude Code for architectural reasoning. This plugin makes that workflow seamless.
- **Book chapter material:** The strategic shift from platform lock-in to ecosystem penetration deserves a chapter section.

## Friction Points

- Dual authentication overhead for teams (Anthropic + OpenAI accounts)
- Split usage limits across platforms
- Background latency — explicit model selection recommended to avoid expensive auto-selection
- Review gate feedback loops can cause cost spikes

## Citations

- [GitHub: openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc)
- [SmartScope: What codex-plugin-cc Means](https://smartscope.blog/en/blog/codex-plugin-cc-openai-claude-code-2026/)
- [The Decoder: OpenAI launches Codex plugin for Claude Code](https://the-decoder.com/openai-launches-a-codex-plugin-that-runs-inside-anthropics-claude-code/)
- [GIGAZINE coverage](https://gigazine.net/gsc_news/en/20260331-claude-code-codex-plugin/)
- [OpenAI Developer Community announcement](https://community.openai.com/t/introducing-codex-plugin-for-claude-code/1378186)

*Published: 2026-03-31*
