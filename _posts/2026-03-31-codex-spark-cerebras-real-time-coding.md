---
title: "GPT-5.3-Codex-Spark and the Cerebras Inference Stack: Real-Time Coding at 1,000 Tokens per Second"
parent: "Articles"
nav_order: 138
tags:
  - models
  - codex-spark
  - model-selection
  - config-toml
  - codex-cli
---

![Sketchnote diagram for: GPT-5.3-Codex-Spark and the Cerebras Inference Stack: Real-Time Coding at 1,000 Tokens per Second](/sketchnotes/articles/2026-03-31-codex-spark-cerebras-real-time-coding.png)

# GPT-5.3-Codex-Spark and the Cerebras Inference Stack: Real-Time Coding at 1,000 Tokens per Second


---

GPT-5.3-Codex-Spark is OpenAI's first model purpose-built for real-time coding iteration, and the first production model served entirely on non-NVIDIA hardware. Released on 12 February 2026 as a research preview for ChatGPT Pro subscribers [^1], Spark trades reasoning depth for raw speed — delivering over 1,000 tokens per second on the Cerebras Wafer-Scale Engine 3 (WSE-3) [^2]. For Codex CLI users, it changes the economics of interactive workflows: rapid prototyping, single-file edits, and frontend iteration become near-instantaneous.

This article covers the hardware underpinnings, the infrastructure optimisations OpenAI shipped alongside Spark, how to configure it in Codex CLI, benchmark realities, and practical patterns for integrating it into a multi-model workflow.

## Model Characteristics at a Glance

GPT-5.3-Codex-Spark is a distilled, smaller derivative of `gpt-5.3-codex` [^12]. It is not a faster version of the same model; it is a separate model checkpoint trained and quantised for the WSE-3 execution profile.

| Property | GPT-5.3-Codex-Spark | GPT-5.3-Codex |
|---|---|---|
| Tokens per second | 1,000+ | 65–70 |
| Context window | 128k | 400k+ |
| Modality | Text-only | Text + images |
| SWE-Bench Pro | ~56% | ~72% |
| Reasoning fields | Not supported | Supported |
| Research preview | ChatGPT Pro only | Generally available |

The accuracy cost of distillation manifests in specific ways during interactive use [^4]:

- Minimal, targeted edits rather than broad architectural rewrites
- No automatic test execution unless explicitly prompted
- "Fast hallucinations that look correct" — fabricated API method names, phantom parameters
- Reduced reliability on structured output formatting

## The Cerebras WSE-3 and Why It Matters

Codex-Spark is the first fruit of OpenAI's multi-year, $10 billion+ agreement with Cerebras Systems [^3]. The WSE-3 is Cerebras' third-generation wafer-scale chip, packing 4 trillion transistors and the largest on-chip memory of any AI accelerator [^2]. The architecture scales out to thousands of systems, pushing fast memory capacity into the multi-terabyte range [^2].

For coding workloads, the key advantage is **latency, not throughput**. Traditional GPU clusters optimise for batch throughput; the WSE-3's on-chip SRAM eliminates the memory-bandwidth bottleneck that causes per-token latency spikes in transformer inference. The result: Spark sustains ~1,000 tokens/s compared to GPT-5.3-Codex's ~65–70 tokens/s — roughly a 15× speed-up [^4].

```mermaid
graph LR
    subgraph "Traditional GPU Inference"
        A[Token Request] --> B[HBM Read]
        B --> C[Compute]
        C --> D[HBM Write]
        D --> E[Next Token]
    end

    subgraph "WSE-3 Inference"
        F[Token Request] --> G[On-Chip SRAM]
        G --> H[Compute + Emit]
    end

    style B fill:#f96,stroke:#333
    style D fill:#f96,stroke:#333
    style G fill:#6f9,stroke:#333
```

## Infrastructure Optimisations

Spark's raw speed would be wasted if the network stack added latency on every round-trip. OpenAI shipped three infrastructure changes alongside the model [^5]:

| Optimisation | Improvement |
|---|---|
| Persistent WebSocket connection | 80% reduction in client/server round-trip overhead |
| Responses API stream rewrite | 30% reduction in per-token overhead |
| Session initialisation rewrite | 50% improvement in time-to-first-token |

The WebSocket transport is enabled by default for Spark and is rolling out as the default for all models [^5]. In `config.toml`, provider-level WebSocket support is exposed via the `supports_websockets` key [^6]:

```toml
[model_providers.openai]
supports_websockets = true
```

## Configuring Spark in Codex CLI

### Quick start

Launch a session with Spark directly:

```bash
codex -m gpt-5.3-codex-spark
```

Or switch mid-session using the `/model` command:

```
/model gpt-5.3-codex-spark
```

### Persistent configuration

Set Spark as your default model in `~/.codex/config.toml`:

```toml
model = "gpt-5.3-codex-spark"
```

Or scope it to a project by placing `.codex/config.toml` in the repository root [^6].

### Reasoning keys to remove

Spark does not implement a chain-of-thought reasoning phase. Any reasoning-related keys in the config must be removed or commented out when switching to Spark [^13]:

```toml
# Remove ALL of these when switching to Spark:
# model_reasoning_effort = "xhigh"
# model_reasoning_summary = "detailed"
# plan_mode_reasoning_effort = "high"
```

Leaving these keys in place does not cause a hard error — they are silently ignored. A config tuned for `gpt-5.3-codex` with reasoning set to `xhigh` will produce confusingly shallow output from Spark even though the session appears to run normally.

### Profile-based switching

For teams that want Spark for rapid iteration but the full model for deep work, profiles provide clean switching [^6]:

```toml
# ~/.codex/config.toml

# Default profile — heavyweight tasks
model = "gpt-5.3-codex"
model_reasoning_effort = "high"
approval_policy = "on-failure"
sandbox_mode = "workspace-write"

# Spark profile — interactive iteration
[profiles.spark]
model = "gpt-5.3-codex-spark"
model_verbosity = "high"
approval_policy = "never"
sandbox_mode = "danger-full-access"

[profiles.spark.features]
multi_agent = true

# Deep profile — complex refactors
[profiles.deep]
model = "gpt-5.3-codex"
model_reasoning_effort = "xhigh"
```

Launch with `codex --profile spark` for quick edits or `codex --profile deep` for complex refactors.

### Fast mode vs. Spark

Codex CLI also exposes `/fast on` for GPT-5.4, which delivers a 1.5x speed increase at 2x credit consumption [^14]. This is a different mechanism from Spark:

- `/fast on` accelerates GPT-5.4 via preferential GPU routing — the same model, faster.
- `gpt-5.3-codex-spark` is a distinct, smaller model on dedicated WSE-3 hardware.

For truly interactive work (sub-second perceived latency), Spark wins. For tasks requiring GPT-5.4's reasoning depth at higher speed, `/fast on` is the appropriate lever.

### Access requirements

Spark is currently available only to ChatGPT Pro subscribers ($200/month) [^1]. API access is rolling out to select design partners [^2]. Users on Plus or Team plans will see the model listed in `/model` but cannot use it — the CLI silently routes to the default model [^7]. Check actual model availability with `/status` rather than `/model` [^8].

Because Spark runs on dedicated WSE-3 hardware, it has its own separate rate limit pool — usage does not count against the standard Codex quota. During periods of high demand, OpenAI may apply additional queuing. Broader access, expanded capabilities (larger models, longer context, multimodal input) are on the roadmap for later in 2026 [^15].

**ChatGPT OAuth caveat:** accessing Codex via ChatGPT OAuth (device-code sign-in) rather than an API key and selecting `gpt-5.3-codex-spark` may return a provider error, as Spark's rate-limit pool is gated separately from the standard ChatGPT backend [^13].

## Benchmark Reality Check

OpenAI's announcement claims Spark matches GPT-5.3-Codex's 77.3% on Terminal-Bench 2.0 [^1]. Independent community benchmarks tell a different story:

| Benchmark | GPT-5.3-Codex | Codex-Spark (OpenAI) | Codex-Spark (Independent) |
|---|---|---|---|
| SWE-Bench Pro | 56.8% [^9] | ~56% ⚠️ | ~56% |
| Terminal-Bench 2.0 | 77.3% [^9] | 77.3% [^1] | ~58.4% [^10] |
| Context Window | 400k+ tokens | 128k tokens | 128k tokens |

The ~19 percentage point gap on Terminal-Bench 2.0 between official and independent scores is significant [^10]. Dominic Elm's analysis notes: "Spark is a smaller, distilled model that trades intelligence for speed, scoring 58.4% on Terminal-Bench 2.0 versus the full Codex's 77.3%" [^10].

⚠️ OpenAI's SWE-Bench Pro claim of "matching" accuracy may reflect a specific scaffold configuration; independent reproductions with standardised scaffolding show lower scores.

The practical takeaway: Spark is meaningfully **more capable than GPT-5.1-Codex-mini** (+12.3 points on Terminal-Bench) [^10] but **not equivalent to the full GPT-5.3-Codex** on complex multi-step tasks. It reportedly drifts after 6–8 reasoning steps, whereas the full model maintains coherence across 12+ step plans [^4].

## When to Use Spark (and When Not To)

```mermaid
flowchart TD
    A[New Task] --> B{Single file edit?}
    B -->|Yes| C{Quick iteration?}
    C -->|Yes| D[Use Spark]
    B -->|No| E{Multi-step reasoning?}
    E -->|Yes| F[Use GPT-5.3-Codex or GPT-5.4]
    E -->|No| G{Security-critical?}
    G -->|Yes| F
    G -->|No| D
    C -->|No| F

    style D fill:#6f9,stroke:#333
    style F fill:#69f,stroke:#333
```

**Spark excels at:**

- Rapid prototyping and boilerplate generation
- Single-file edits (CSS, React components, configuration)
- Frontend iteration cycles
- Interactive pair programming where latency matters more than depth
- Codebase queries and exploration — "which file owns X?" style questions
- Plan revision in an active planning session where the plan is already established

A useful heuristic: if the result can be verified in under 30 seconds (visual check, quick test run), Spark is a good fit.

**Stick with the full model for:**

- Multi-step debugging across services
- Security-critical code (authentication, encryption, validation)
- Database migrations requiring stateful reasoning
- Large codebase analysis beyond the 128k context window
- Multi-file refactors touching 5+ files
- Long planning sessions requiring 12+ reasoning steps
- Structured output that must conform to a strict schema (Spark's reliability here is lower)

## Multi-Model Workflow: Spark as a Subagent

One powerful pattern combines Spark's speed with a flagship model's reasoning. Using Codex CLI's multi-agent v2 system, you can delegate quick edits to a Spark-powered subagent while the primary agent handles orchestration:

```toml
# .codex/agents/quick-edit.toml
model = "gpt-5.3-codex-spark"
model_reasoning_effort = "low"
instructions = """
You handle single-file edits and quick formatting fixes.
Keep changes minimal and focused.
"""
```

The primary agent (running GPT-5.4 or GPT-5.3-Codex) can then spawn this subagent for rapid tasks while retaining the full model's reasoning for architectural decisions.

### Spark for live pairing

Spark's latency profile matches human thinking speed. During active editing sessions where the model should feel like fast autocomplete rather than a batch job:

```bash
# Start a Spark session for current-file work
codex --profile spark "Refactor this function to use early returns instead of nested conditionals"
```

### Hybrid task decomposition

Breaking tasks explicitly before dispatching enables mixing models within a single workflow:

```
1. [SPARK] Add the new UserPreferences interface to types/user.ts
2. [SPARK] Update the three component files that consume UserPreferences
3. [CODEX] Write migration script to backfill the new preference column
4. [CODEX] Review the full changeset for security implications
```

Steps 1 and 2 are single-file and easily verifiable — Spark finishes each in seconds. Steps 3 and 4 involve multi-file reasoning and correctness guarantees where the full model is warranted.

## What the Speed Actually Unlocks

The qualitative shift at 1,000 tokens per second is not merely "faster waiting." It changes the interaction model:

- **Speculation becomes cheap.** At 65 tokens/second, trying five different approaches to a function signature is a five-minute commitment. At 1,000 tokens/second, it takes thirty seconds. Experimentation increases.
- **Context stays warm.** The cognitive overhead of re-establishing context after a long wait largely disappears when responses arrive before the developer loses their train of thought.
- **Shorter prompts work better.** Because iteration is fast, sending a rough prompt and correcting the output is often more efficient than crafting an exhaustive specification upfront.

As Simon Willison observed: "When a model responds this fast you can stay in flow state and iterate with the model much more productively" [^16]. The tradeoff is that this flow state operates with a model at ~56% SWE-Bench Pro accuracy — adequate for interactive iteration, inadequate for unattended agentic tasks.

## API Pricing and Cost Considerations

For API access (rolling out to design partners), Spark's pricing is significantly lower than the full model [^11]:

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|---|---|---|
| gpt-5.3-codex-spark | $1.75 | $14.00 |

⚠️ API pricing is subject to change as Spark exits research preview.

Combined with its 15× throughput, Spark can be dramatically more cost-effective for high-volume, low-complexity coding tasks — particularly in CI/CD pipelines using `codex exec` for automated formatting, linting fixes, and boilerplate generation.

## The Strategic Picture

Codex-Spark represents OpenAI's first step toward a dual-mode Codex architecture: real-time collaboration for rapid iteration alongside longer-horizon reasoning for deep work [^2]. Over time, these modes will blend — Codex maintaining an interactive loop while delegating complex work to background subagents [^2].

For the Codex CLI ecosystem, the implications are clear: model selection is no longer just about capability but about **matching latency characteristics to task complexity**. The profile system, multi-agent delegation, and the upcoming plugin marketplace all support this heterogeneous-model future.

The Cerebras partnership also signals OpenAI's strategic diversification away from NVIDIA dependency [^3]. Whether WSE-3 inference scales to flagship models remains to be seen, but for distilled, speed-optimised variants like Spark, the wafer-scale approach delivers a qualitatively different developer experience.

## Citations

[^1]: OpenAI, "Introducing GPT-5.3-Codex-Spark," 12 February 2026. <https://openai.com/index/introducing-gpt-5-3-codex-spark/>

[^2]: Cerebras, "Introducing OpenAI GPT-5.3-Codex-Spark Powered by Cerebras," February 2026. <https://www.cerebras.ai/blog/openai-codexspark>

[^3]: Tom's Hardware, "OpenAI launches GPT-5.3-Codex-Spark on Cerebras chips — marks AI giant's first production deployment away from Nvidia," February 2026. <https://www.tomshardware.com/tech-industry/artificial-intelligence/openai-lauches-gpt-53-codes-spark-on-cerebras-chips>

[^4]: Turing College, "Codex 5.3 vs. Codex Spark: Speed vs. Intelligence," 2026. <https://www.turingcollege.com/blog/codex-5-3-vs-codex-spark-speed-vs-intelligence>

[^5]: InfoQ, "OpenAI Codex-Spark Achieves Ultra-Fast Coding Speeds on Cerebras Hardware," March 2026. <https://www.infoq.com/news/2026/03/open-ai-codex-spark/>

[^6]: OpenAI, "Configuration Reference – Codex," 2026. <https://developers.openai.com/codex/config-reference>

[^7]: GitHub, "Codex Spark not yet available? · Issue #11623," 12 February 2026. <https://github.com/openai/codex/issues/11623>

[^8]: GitHub, "Limitations of GPT-5.3-Codex-Spark visible in /status but not /model · Issue #12992," 2026. <https://github.com/openai/codex/issues/12992>

[^9]: OpenAI, "Introducing GPT-5.3-Codex," 5 February 2026. <https://openai.com/index/introducing-gpt-5-3-codex/>

[^10]: Dominic Elm (@elmd_), "GPT-5.3-Codex-Spark is not GPT-5.3-Codex!," X/Twitter, February 2026. <https://x.com/elmd_/status/2023417837193240788>

[^11]: TypingMind, "Connect and use GPT-5.3 Codex Spark from OpenAI with API Key," 2026. <https://www.typingmind.com/guide/openai/gpt-5.3-codex-spark>

[^12]: Simon Willison, "Introducing GPT-5.3-Codex-Spark," February 2026. <https://simonwillison.net/2026/Feb/12/codex-spark/>

[^13]: XAI Router, "GPT-5.3 Codex Spark: Faster, But You Should Not Reuse gpt-5.3-codex Config," 2026. <https://xairouter.com/en/blog/gpt-5-3-codex-spark/>

[^14]: OpenAI, "Speed – Codex," 2026. <https://developers.openai.com/codex/speed>

[^15]: eWeek, "OpenAI Debuts GPT-5.3-Codex-Spark, a Near-Instant AI for Real-Time Coding," 2026. <https://www.eweek.com/news/openai-gpt-5-3-codex-spark-real-time-coding-cerebras/>

[^16]: Simon Willison ibid. — flow state observation on sub-second model response times.
