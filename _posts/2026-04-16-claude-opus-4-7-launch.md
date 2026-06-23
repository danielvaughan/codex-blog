---
title: "Claude Opus 4.7 Launch: What It Means for AI Coding Agents"
description: "Published: 16 April 2026 Source: anthropic.com/news/claude-opus-4-7"
date: 2026-04-16T00:00:00+00:00
last_modified_at: 2026-06-23T07:07:37+01:00
---

![Sketchnote diagram for: Claude Opus 4.7 Launch: What It Means for AI Coding Agents](/sketchnotes/articles/2026-04-16-claude-opus-4-7-launch.png)

# Claude Opus 4.7 Launch: What It Means for AI Coding Agents

**Source:** [anthropic.com/news/claude-opus-4-7](https://www.anthropic.com/news/claude-opus-4-7)

---

## Overview

On 16 April 2026, Anthropic released Claude Opus 4.7 (API identifier: `claude-opus-4-7`), the successor to Opus 4.6. Pricing remains unchanged at \$5 per million input tokens and \$25 per million output tokens -- the same as Opus 4.6 -- making it a capability upgrade rather than a pricing-tier shift.

Opus 4.7 is available on Claude.ai, the Anthropic API, Amazon Bedrock, Google Cloud Vertex AI, and Microsoft Foundry.

---

## Key Improvements

### Improved Agentic Execution Rigor

Opus 4.7 introduces tighter agentic execution guarantees. The model demonstrates improved self-verification capabilities, meaning it is more likely to check its own work before declaring a task complete. For coding agents like Claude Code and Codex CLI (via cross-model bridging), this translates to fewer silent failures in multi-step workflows and better adherence to instructions that require sequential reasoning across tool calls.

### Enhanced Vision: 3.75 MP (2,576 px Long Edge)

The maximum image resolution jumps to 3.75 megapixels with a 2,576-pixel long edge -- approximately 3x larger than prior Opus models. This is significant for workflows that involve screenshot analysis, UI testing, design-to-code pipelines, and diagram comprehension. Agents processing visual inputs will get substantially more detail per image without needing to tile or downscale.

### New `xhigh` Effort Level

Opus 4.7 adds a new reasoning effort level called `xhigh`, positioned between `high` and `max`. This gives developers finer control over the cost-quality tradeoff for complex tasks that benefit from extended reasoning but do not require the full token budget of `max` effort.

### Updated Tokenizer

The tokenizer has been updated, resulting in a 1.0-1.35x token increase for equivalent prompts compared to Opus 4.6. Teams should factor this into cost projections and context window management, particularly for long-session workflows where token budgets are tight.

### New `/ultrareview` Slash Command in Claude Code

Opus 4.7 ships alongside a new `/ultrareview` slash command in Claude Code that leverages the model's improved self-verification capabilities for deep code review passes. This extends the existing review workflow with a more thorough multi-pass analysis.

---

## Availability

| Platform | Status |
|---|---|
| Claude.ai | Available |
| Anthropic API | Available (`claude-opus-4-7`) |
| Amazon Bedrock | Available |
| Google Cloud Vertex AI | Available |
| Microsoft Foundry | Available |

---

## Pricing

| Tier | Input | Output |
|---|---|---|
| Opus 4.7 | \$5 / MTok | \$25 / MTok |
| Opus 4.6 (unchanged) | \$5 / MTok | \$25 / MTok |

No pricing change from the previous generation.

---

## Benchmarks

Early benchmark data shows Opus 4.7 delivering measurable improvements across multiple domains:

| Benchmark | Opus 4.7 | Opus 4.6 | Improvement |
|-----------|----------|----------|-------------|
| Coding (93-task eval) | — | — | +13% lift |
| CursorBench | 70% | 58% | +12 pts |
| General Finance | 0.813 | 0.767 | +6% |
| Visual acuity | 98.5% | 54.5% | +44 pts |
| Document reasoning | — | — | 21% fewer errors |
| Production tasks (Rakuten) | 3× more completed | baseline | +200% throughput |
| Tool-call accuracy | — | — | Double-digit improvement |

Anthropic claims Opus 4.7 beats ChatGPT 5.4 and Gemini 3.1 Pro across key benchmarks. The tool-call accuracy improvement and failure recovery capability are particularly relevant for agentic workflows — agents that can recover from tool failures mid-execution waste fewer tokens on retries and dead-end paths.

### Mythos Preview Context

Anthropic concedes that Opus 4.7 still trails its unreleased **Mythos Preview** model, which has been shared only with select tech and cybersecurity companies. According to [Axios](https://www.axios.com/2026/04/16/anthropic-claude-opus-model-mythos), Opus 4.7 includes differential training to reduce cyber capabilities compared to Mythos — a deliberate capability cap for the public release. A new **Cyber Verification Program** enables legitimate security researchers to access fuller capabilities for vulnerability testing and red-teaming.

---

## Impact on Codex CLI and Claude Code Workflows

For Claude Code users, Opus 4.7 is a direct upgrade: better agentic execution, larger vision inputs, and the new `/ultrareview` command. Auto Mode is now extended to Max users for autonomous decision-making. For Codex CLI users who access Claude models via the `codex-plugin-cc` cross-model bridge or multi-provider configurations, the improved self-verification and vision capabilities carry over to cross-platform workflows.

The tokenizer change (1.0-1.35x increase) is the main operational consideration. Teams running close to context window limits or with tight per-session token budgets should test their existing workflows against Opus 4.7 before switching, and adjust compaction thresholds or budget caps accordingly.

### Task Budgets (Public Beta)

Opus 4.7 introduces **task budgets** in public beta — a feature to guide token spending on longer runs. This directly parallels Codex CLI's own goal mode token budgets (PRs #18074-#18077), suggesting convergent evolution: both platforms are solving the same problem of autonomous agents needing spending guardrails.

---

## Competitive Implications

| Dimension | Codex CLI (GPT-5.4) | Claude Code (Opus 4.7) |
|-----------|---------------------|------------------------|
| Coding benchmark | Strong (GPT-5.4 Codex line) | 70% CursorBench, +13% coding lift |
| Tool failure recovery | Guardian + hooks | Native model capability |
| Token cost control | Goal mode budgets | Task budgets (beta) |
| Effort levels | Standard reasoning | `xhigh` new tier |
| Autonomy | Goal mode, full-auto | Auto Mode (93% approval) |
| Vision | Screenshot via MCP | 3.75 MP native (2,576 px) |

The competitive gap is narrowing on the autonomy axis: both platforms are converging on "supervised autonomous" modes with cost guardrails. The key differentiator remains infrastructure — Codex CLI's Rust-based sandbox and hook system vs Claude Code's model-native self-verification.

---

## Further Reading

- [Anthropic announcement](https://www.anthropic.com/news/claude-opus-4-7)
- [Axios: Anthropic releases Claude Opus 4.7, concedes it trails Mythos](https://www.axios.com/2026/04/16/anthropic-claude-opus-model-mythos)
- [9to5Mac: New Opus 4.7 model with focus on advanced software engineering](https://9to5mac.com/2026/04/16/anthropic-reveals-new-opus-4-7-model-with-focus-on-advanced-software-engineering/)
- [GitHub Blog: Claude Opus 4.7 is generally available](https://github.blog/changelog/2026-04-16-claude-opus-4-7-is-generally-available/)
- Chapter 4 (Benchmarks) and Chapter 5 (Competing Tools) of the Codex CLI book cover Claude model performance in detail
