# Claude Opus 4.7 Launch: What It Means for AI Coding Agents

**Published:** 16 April 2026
**Source:** [anthropic.com/news/claude-opus-4-7](https://www.anthropic.com/news/claude-opus-4-7)

---

## Overview

On 16 April 2026, Anthropic released Claude Opus 4.7 (API identifier: `claude-opus-4-7`), the successor to Opus 4.6. Pricing remains unchanged at $5 per million input tokens and $25 per million output tokens -- the same as Opus 4.6 -- making it a capability upgrade rather than a pricing-tier shift.

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
| Opus 4.7 | $5 / MTok | $25 / MTok |
| Opus 4.6 (unchanged) | $5 / MTok | $25 / MTok |

No pricing change from the previous generation.

---

## Impact on Codex CLI and Claude Code Workflows

For Claude Code users, Opus 4.7 is a direct upgrade: better agentic execution, larger vision inputs, and the new `/ultrareview` command. For Codex CLI users who access Claude models via the `codex-plugin-cc` cross-model bridge or multi-provider configurations, the improved self-verification and vision capabilities carry over to cross-platform workflows.

The tokenizer change (1.0-1.35x increase) is the main operational consideration. Teams running close to context window limits or with tight per-session token budgets should test their existing workflows against Opus 4.7 before switching, and adjust compaction thresholds or budget caps accordingly.

---

## Further Reading

- [Anthropic announcement](https://www.anthropic.com/news/claude-opus-4-7)
- Chapter 4 (Benchmarks) and Chapter 5 (Competing Tools) of the Codex CLI book cover Claude model performance in detail
