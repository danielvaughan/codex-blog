---
title: "Codex Model Lineage: The Context Compaction Breakthrough That Made Long-Horizon Agents Possible"
date: 2026-03-28
tags: [model-lineage, context-compaction, gpt-5-2-codex, gpt-5-3-codex, long-horizon, benchmarks]
summary: "How the GPT-5.x-Codex model family evolved to solve the context problem — and why native compaction changed what agentic coding workflows can actually achieve."
---
![Sketchnote: Codex Model Lineage: The Context Compaction Breakthrough That Made Long-Horizon Agents Possible](/sketchnotes/articles/2026-03-28-codex-model-lineage-context-compaction.png)

# Codex Model Lineage: The Context Compaction Breakthrough

*Published: 2026-03-28 | Sources: [openai.com/index/introducing-gpt-5-2-codex](https://openai.com/index/introducing-gpt-5-2-codex/), [openai.com/index/introducing-gpt-5-3-codex](https://openai.com/index/introducing-gpt-5-3-codex/)*

---

If you've been using Codex CLI for agentic work, you've probably hit the wall: a long-running session starts hallucinating earlier decisions, loses track of what files were changed, and forces you to restart. This isn't a prompting problem — it's a context architecture problem. Understanding how the Codex model family evolved to address it reveals *why* modern long-horizon workflows are now actually viable.

---

## The GPT-5.x-Codex Family at a Glance

| Model | Released | Key advance | SWE-Bench Pro | Terminal-Bench 2.0 |
|-------|----------|-------------|--------------|-------------------|
| `gpt-5.1-codex-max` | Late 2025 | Compaction *across* windows (external scaffolding) | ~50.8% | ~58.1% |
| `gpt-5.2-codex` | Jan 14, 2026 | **Native** context compaction baked into model | 56.4% | 64.0% |
| `gpt-5.3-codex` | Feb 5, 2026 | Frontier reasoning + coding unified; interactive steering | 56.8% | 77.3% |
| `gpt-5.3-codex-spark` | Feb 2026 | 1,000+ tok/s on Cerebras hardware (research preview) | — | — |
| `gpt-5-codex` | March 2026 | New flagship; 4x efficiency sibling (gpt-5-codex-mini) | — | — |

---

## What Changed in GPT-5.2-Codex

**Released: January 14, 2026** ([OpenAI announcement](https://openai.com/index/introducing-gpt-5-2-codex/))

GPT-5.2-Codex is a fine-tune of GPT-5.2 purpose-built for professional software engineering and defensive cybersecurity. Its headline capability: **native context compaction**.

### Before: compaction as external scaffolding

GPT-5.1-Codex-Max could work across multiple context windows — but the compaction was handled by external harness code: summarise older turns, inject the summary, continue. This worked but introduced its own failure modes: the harness might summarise poorly, the model wasn't trained to expect or interpret those summaries, and long-range coherence suffered.

### After: compaction baked into the model itself

GPT-5.2-Codex was specifically optimised for Codex's compaction workflow. The model knows, during training, that context windows compress and resume. It produces and consumes summaries that are semantically faithful yet token-efficient. The result: **genuinely coherent multi-file sessions** that span thousands of file operations without losing track.

> *"The ability for the model to work coherently across multiple context windows"* — OpenAI, on how GPT-5.2-Codex's cybersecurity CTF jump was attributed to compaction.

---

## Key Benchmarks

- **SWE-Bench Pro: 56.4%** — evaluates correct patches in large, unfamiliar codebases (↑ from 50.8% for GPT-5.1)
- **Terminal-Bench 2.0: 64.0%** — live terminal environment agentic behaviour
- **CVE-Bench: 87%** — cybersecurity vulnerability identification and patching
- **Tau2-bench tool-call accuracy: 98.7%** — agentic workflows rarely fail due to incorrect tool invocations (critical for reliability in automation pipelines)
- **SWE-Bench Verified: 80%**

The Tau2-bench number is under-discussed. A 98.7% tool-call accuracy rate means that in an automated CI pipeline, the Codex agent very rarely issues malformed tool calls that break the workflow. This is what makes `codex exec` in unattended pipelines actually practical.

---

## The Architecture: MoE Matters

GPT-5.2-Codex uses a **Mixture-of-Experts (MoE) architecture** with sparse activation. Only a subset of parameters fire per token, which:

1. Keeps per-token inference costs manageable despite deep specialisation
2. Allows specialised "expert" sub-networks for coding, tool use, and cybersecurity to operate independently
3. Enables the model to be both broad (Tau2-bench tool accuracy) and deep (CVE-Bench security)

This is why the pricing ($1.75/M input, $14/M output) is reasonable for a model that performs at enterprise coding standards.

---

## What GPT-5.3-Codex Added

Released February 5, 2026, GPT-5.3-Codex extends the compaction foundation with two additions:

1. **Interactive steering without context loss** — you can redirect the agent mid-task and it maintains coherent state. Previously, mid-task redirects often caused the agent to lose its thread.
2. **Frontier reasoning unified into the coding model** — GPT-5.3-Codex combines the coding performance of GPT-5.2-Codex with the reasoning depth of GPT-5.2 base in a single system, 25% faster.

Terminal-Bench 2.0 score jumps to **77.3%** (from 64.0%) — a 13 percentage point gain that reflects improved tool use chaining and long-horizon task completion, not just code generation quality.

---

## Practical Implications for Agentic Workflows

### 1. Long autonomous runs are now viable defaults

With GPT-5.2-Codex and later, the `/compact` command became a genuine strategy (not a workaround). The model is trained to handle compacted context — so proactively compacting at 60% utilisation actually works as intended. Earlier models lost coherence post-compaction more readily.

### 2. CI/CD automation unlocked by 98.7% tool reliability

For `codex exec` in unattended GitHub Actions pipelines, the bottleneck was previously incorrect tool invocations breaking the automation. GPT-5.2-Codex's Tau2-bench result suggests this failure mode is largely solved.

### 3. Subagent economics changed

With higher per-subagent reliability, you can now afford to spin up more subagents per orchestration run without expecting a fixed failure rate. The 56.4% SWE-Bench Pro performance means each subagent resolves harder tasks before needing human escalation.

### 4. Cybersecurity as a first-class use case

The jump to 87% on CVE-Bench makes GPT-5.2-Codex (and its successors) viable for automated security review in the agentic pod. Codex Security (launched March 21, 2026) uses this capability lineage.

---

## Model Selection Guidance (Updated)

```toml
# config.toml — recommended routing strategy
[profiles.explore]
model = "gpt-5-codex-mini"       # fast exploration, cost-efficient subagents

[profiles.commit]
model = "gpt-5-codex"            # standard commits, most tasks

[profiles.deep]
model = "gpt-5.3-codex"          # complex long-horizon sessions, security reviews

[profiles.legacy-long]
model = "gpt-5.2-codex"          # still available; good for predictable compaction-heavy runs
```

If you have existing workflows built around GPT-5.2-Codex (e.g., pinned in CI), they remain fully supported and the compaction behaviour is stable. Upgrading to GPT-5.3-Codex gives you interactive steering; upgrading to `gpt-5-codex` gives you the new efficiency tier.

---

## What to Watch in v0.118.0

The v0.118.0 alpha builds (alpha.1–alpha.3 as of March 27) haven't published release notes yet. Given the model lineage trajectory, watch for:
- Further compaction improvements (multi-agent session handoff)
- `gpt-5-codex` becoming the default model
- Spark model graduating from research preview

*See: [notes/changelog-watch.md](/codex-resources/notes/changelog-watch/) for v0.118.0 tracking*

---

*Added: 2026-03-28 | Covers: GPT-5.2-Codex (Jan 14, 2026), GPT-5.3-Codex (Feb 5, 2026)*
