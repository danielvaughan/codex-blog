---
title: "Gemma 4's Flash Attention Freeze: What It Means for Local AI Coding"
date: 2026-04-12T14:00:00+00:00
tags:
  - gemma-4
  - local-models
  - ollama
  - flash-attention
  - codex-cli
  - troubleshooting
---

![Sketchnote diagram for: Gemma 4's Flash Attention Freeze: What It Means for Local AI Coding](/sketchnotes/articles/2026-04-12-gemma-4-flash-attention-freeze-local-ai-coding.png)

Google's Gemma 4 launched this week to genuine excitement. A 31B dense model and a 26B mixture-of-experts variant, both open-weight, both promising flagship-tier reasoning at local inference costs. For developers running Codex CLI with local models via Ollama, this was supposed to be the moment local AI coding became viable for serious work. Instead, a Flash Attention bug is freezing machines across both NVIDIA and Apple Silicon hardware — and the workarounds come with painful trade-offs.

## The Bug

The issue is specific to Flash Attention (FA) and Gemma 4's hybrid attention architecture[^1][^2]. Unlike most transformer models that use uniform attention heads, Gemma 4 combines 50 sliding-window attention layers with 10 global attention layers, each using different head dimensions — 256 for sliding-window, 512 for global. The Flash Attention implementation in Ollama does not handle this dual-dimension layout correctly.

The symptoms are dramatic. On NVIDIA hardware (RTX 3090, RTX 4090), the 31B Dense model hangs indefinitely during prompt evaluation when prompts exceed approximately 3-4K tokens[^1]. GPU utilisation drops to 0%. There is no error message, no timeout, no graceful failure — just a complete stall. Short prompts work perfectly at full speed, making the bug particularly insidious: everything appears fine until you try to do real work with a substantial system prompt.

On Apple Silicon (M4, M5 Max), the threshold is even lower — around 500 tokens before the hang occurs[^2]. On an M5 Max with 128GB unified memory, processing a 31K-token prompt without Flash Attention takes approximately 190 seconds. With Flash Attention enabled, it never completes.

The 26B MoE variant is partially affected too. While it handles large prompts better than the Dense model in some configurations, the doubled data volume from the MoE architecture triggers its own FA failures[^2].

## Why This Matters for Codex CLI Users

Local model inference is becoming a real part of the Codex CLI ecosystem. The model routing patterns covered in our earlier article on [advisor and routing strategies](/2026/04/12/model-routing-advisor-patterns-cut-ai-coding-costs/) show developers using local models as Tier 0 for classification, routing, and summarisation tasks. Codex CLI's profile system supports pointing at local Ollama instances, and the 4-tier routing architecture depends on cheap local inference for background operations.

Gemma 4 was the model that was supposed to make this practical. The 26B MoE variant fits in 16GB of VRAM (quantised), runs multimodal tasks, and benchmarks competitively with GPT-4o on reasoning tasks. But agentic coding workflows typically involve system prompts of 4-8K tokens (AGENTS.md, project context, tool definitions) — exactly the range where the freeze occurs.

This means Codex CLI users attempting to use Gemma 4 as a local model via Ollama will hit the freeze on virtually every real-world coding task.

## The Workarounds (and Their Costs)

### 1. Disable Flash Attention

```bash
OLLAMA_FLASH_ATTENTION=0 ollama serve
```

This is the primary workaround. It eliminates the freeze entirely but at significant performance cost[^1][^2]:

- **NVIDIA RTX 3090**: Generation speed drops substantially without FA optimisations
- **Apple Silicon M5 Max**: ~15 tok/s generation, ~168 tok/s prompt eval on the 31B Dense
- **Apple Silicon M5 Max**: ~75 tok/s on the 26B MoE without FA

For interactive coding, 15 tok/s is borderline unusable. For background agent tasks where latency matters less, it is workable but slow.

### 2. Use the 26B MoE Instead of 31B Dense

The MoE variant processes large prompts more reliably in most configurations and runs faster due to only activating 4B parameters per forward pass[^1]:

```bash
ollama run gemma4:26b
```

This is the pragmatic choice for most Codex CLI users. The 26B MoE with FA disabled at ~75 tok/s on Apple Silicon is usable for Tier 0 routing tasks.

### 3. Keep System Prompts Below the Threshold

If you must use the 31B Dense with FA enabled, keep total prompt tokens under 3K (NVIDIA) or 500 (Apple Silicon). This is impractical for most agentic workflows but may work for simple classification tasks with minimal context.

### 4. Use vLLM Instead of Ollama

vLLM has its own issues — Gemma 4 forces a Triton attention fallback yielding only ~9 tok/s on an RTX 4090[^3] — but some configurations handle large prompts without the complete freeze. This trades one set of problems for another.

### 5. Wait for the Fix

The Ollama team is aware of the issue. The root cause is understood (FA kernel needs to handle heterogeneous head dimensions), but no timeline for a fix has been published[^1][^2].

## The Broader Pattern

This is not the first time a promising open-weight model has launched with inference infrastructure issues. Gemma 3 27B had similar FA problems that took weeks to resolve. The pattern is becoming predictable: model team ships weights, inference frameworks scramble to support new architectures, edge cases in attention implementations surface under real workloads.

For Codex CLI users building local model pipelines, the lesson is clear: **test your local model setup with production-length prompts before committing to it in your routing configuration.** A model that works perfectly on short test prompts may freeze completely when it encounters your actual AGENTS.md plus project context plus tool definitions.

The Gemma 4 architecture is genuinely impressive. The hybrid attention design that causes the FA bug is actually an innovation — mixing sliding-window efficiency with global attention capability. The bug is in the inference implementation, not the model. When the fix lands, Gemma 4 will likely become the default local model for Codex CLI's Tier 0 routing. Until then, run with `OLLAMA_FLASH_ATTENTION=0` and use the 26B MoE variant.

## Codex CLI Configuration

For Codex CLI users who want to use Gemma 4 locally today, here is a working profile configuration:

```toml
[profiles.local]
model = "gemma4:26b"
provider = "ollama"
approval_policy = "never"
model_reasoning_effort = "low"
```

And the Ollama launch command:

```bash
OLLAMA_FLASH_ATTENTION=0 OLLAMA_KV_CACHE_TYPE=q8_0 ollama serve
```

This gives you a functional local Tier 0 model for classification, routing, and summarisation tasks while avoiding the freeze entirely.

---

## Citations

[^1]: GitHub, [ollama/ollama#15350: Gemma 4 31B Dense — Flash Attention hangs indefinitely on large prompt eval (>3-4K tokens)](https://github.com/ollama/ollama/issues/15350). NVIDIA RTX 3090, Ubuntu 24.04, Ollama v0.20.2. Progressive per-batch slowdown during prompt eval.

[^2]: GitHub, [ollama/ollama#15368: Gemma 4 on Apple Silicon M5 Max — FA hang, /v1 streaming reasoning field, MLX not supported](https://github.com/ollama/ollama/issues/15368). MacBook Pro M5 Max 128GB. Three bugs: FA hang >500 tokens, /v1 streaming broken, MLX unsupported.

[^3]: GitHub, [vllm-project/vllm#38887: Gemma 4 E4B extremely slow on v0.19.0](https://github.com/vllm-project/vllm/issues/38887). Forced Triton attention fallback yields ~9 tok/s on RTX 4090.
