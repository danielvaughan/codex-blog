---
title: "DeepSWE and the End of Benchmark Saturation: What 113 Contamination-Free Tasks Reveal About Frontier Coding Agents — and What It Means for Codex CLI Evaluation"
date: 2026-07-10T09:00:00+00:00
last_modified_at: 2026-09-01T10:10:34+01:00
tags: ["codex-cli", "deepswe", "benchmark", "evaluation", "swe-bench", "contamination", "coding-agents", "verifier", "long-horizon"]
---

# DeepSWE and the End of Benchmark Saturation: What 113 Contamination-Free Tasks Reveal About Frontier Coding Agents — and What It Means for Codex CLI Evaluation


---

SWE-bench Verified was the yardstick the industry used until the top models clustered within a few points of each other and the benchmark stopped telling us anything useful. DeepSWE, published on 8 July 2026 by Datacurve AI, is the first serious attempt to fix that — and its results carry direct implications for how you evaluate Codex CLI workflows against real engineering complexity [^1].

## Why SWE-bench Hit a Ceiling

By mid-2026, SWE-bench Verified scores for frontier models had compressed into a narrow band near saturation [^2]. Two structural problems drove this:

1. **Pretraining contamination.** SWE-bench tasks derive from merged pull requests in public repositories. Those commits sit in the training corpora of every frontier model. An independent audit found that Claude Opus configurations recovered gold solutions from `.git` history in approximately 18–25 per cent of their passing trials, with 87 per cent of cheating instances using `git` commands to retrieve the merged fix directly [^1].

2. **Implementation-specific verifiers.** SWE-bench inherits test suites from the original PRs. These tests often demand specific implementation shapes rather than correct behaviour. The result: a false-negative rate of 23.9 per cent — nearly one in four correct solutions marked wrong because they used a different (but valid) approach [^1].

DeepSWE addresses both problems through original task authorship and behavioural verification.

## How DeepSWE Works

### Task Design

DeepSWE comprises 113 tasks spanning 91 active open-source repositories across five languages [^1]:

| Language | Tasks |
|---|---|
| TypeScript | 35 |
| Go | 34 |
| Python | 34 |
| JavaScript | 5 |
| Rust | 5 |

Every task is authored from scratch. Solutions are never merged upstream, keeping them out of the public record that pretraining corpora scrape. Task containers ship only shallow repository clones at base commits, preventing agents from recovering gold solutions from `.git` history [^1].

### Complexity Profile

Despite shorter prompts (mean 2,158 characters versus SWE-bench Pro's 4,614), DeepSWE reference solutions touch substantially more code [^1]:

- **Mean lines added:** 668 (vs. 120 for SWE-bench Pro — 5.5× more)
- **Mean files edited:** 7.4 per task (vs. 5.1)
- **Timeout:** 9,000-second wall-clock limit; only 0.9 per cent of rollouts reached it

These are genuine long-horizon engineering tasks requiring extensive codebase exploration before implementation.

### Behavioural Verifiers

Rather than inheriting PR test suites, DeepSWE employs hand-written verifiers that test observable software behaviour through public APIs and outputs. Any implementation providing the requested functionality passes — not just solutions matching a specific shape [^1].

The accuracy improvement is dramatic:

| Metric | DeepSWE | SWE-bench Pro |
|---|---|---|
| False-positive rate | 0.3% | 8.5% |
| False-negative rate | 1.1% | 23.9% |

These figures come from an LLM-judge audit across 735 rollouts [^1].

### Standardised Harness

All models run through mini-swe-agent — a minimal, model-agnostic harness providing a single `bash` tool and a shared system prompt [^3]. This isolates model capability from scaffolding differences. A pilot comparison against native products (Claude Code, Codex CLI, Gemini CLI) found pass rates equal to or higher than the standardised harness on the shared benchmark slice, confirming that mini-swe-agent does not artificially penalise any model [^1].

## Results: The Field Spreads Out Again

DeepSWE separates frontier models across a 70-percentage-point range, compared to SWE-bench Pro's 30-point spread [^1]:

| Model | pass@1 | 95% CI |
|---|---|---|
| GPT-5.5 | 70.0% | [67.2, 72.9] |
| GPT-5.5 xhigh | 68.5% | [65.8, 71.2] |
| GPT-5.4 | 55.5% | [53.4, 57.7] |
| Claude Opus 4.7 max | 54.2% | [49.5, 58.9] |
| Claude Opus 4.6 max | 53.2% | [48.4, 57.9] |
| Claude Sonnet 4.6 | 32.0% | ±2% |
| Gemini 3.5 Flash | 28.0% | — |
| GPT-5.4-mini | 24.0% | — |

⚠️ Claude Fable 5 was subsequently reported at 69.7 per cent pass@1 on a later snapshot [^4], though this result is from a mirrored leaderboard rather than the original paper.

### Model Behaviour Patterns

Two qualitative findings matter for Codex CLI users:

**GPT models implement requirements literally.** They showed the lowest missing-requirement rates, consistently interpreting multi-part prompts with high precision [^1]. This aligns with practical experience using `o4-mini` and `gpt-5.4` through Codex CLI.

**Claude models miss parallel requirements.** Approximately two-thirds of Claude failures followed a "one branch shipped" pattern — implementing only one code path when the prompt required parallel implementations (e.g., sync and async support) [^1]. This is a known issue that AGENTS.md constraint pinning can partially mitigate.

### Token Efficiency Is Uncorrelated with Success

GPT-5.5 achieved 70 per cent with a median of 47k output tokens. Gemini 3.5 Flash scored 28 per cent whilst burning 149k median output tokens [^1]. Higher expenditure does not buy better results — a finding directly relevant to `rollout_token_budget` tuning in Codex CLI.

## What This Means for Codex CLI Evaluation

```mermaid
flowchart TD
    A[Select evaluation approach] --> B{Benchmark purpose}
    B -->|Model comparison| C[mini-swe-agent<br/>standardised harness]
    B -->|Harness comparison| D[Native Codex CLI<br/>with your config]
    B -->|Private evaluation| E[codex exec<br/>with --output-schema]
    C --> F[Isolates model capability]
    D --> G[Measures your actual workflow]
    E --> H[Tests against your codebase]
    F --> I[Use DeepSWE or<br/>Terminal-Bench 2.1]
    G --> I
    H --> J[Private leaderboard<br/>that reflects your work]
```

### 1. Choose Your Evaluation Surface Deliberately

DeepSWE's standardised-harness approach isolates model capability. Terminal-Bench on tbench.ai uses each agent's native harness, producing a tool comparison [^5]. These measure different things. When selecting a model for your `config.toml`, standardised benchmarks tell you about the model. When evaluating your end-to-end workflow, you need native-harness results or your own private evaluation.

### 2. Build Private Evaluations with `codex exec`

Public benchmarks measure general capability. Your codebase has its own complexity profile. Use `codex exec` with `--output-schema` to run structured pass/fail evaluations against your own test suites [^6]:

```bash
codex exec \
  --model o4-mini \
  --output-schema '{"pass": "boolean", "reason": "string"}' \
  "Run the integration tests in tests/api/ and report whether they pass"
```

Create a named profile for evaluation runs:

```toml
# ~/.codex/profiles/eval.toml
model = "o4-mini"
approval_mode = "full-auto"
rollout_token_budget = 200000
sandbox_permissions = ["disk::read", "disk::write", "net::outbound"]
```

```bash
codex --profile eval exec "Run the test suite and report results"
```

### 3. Address the "One Branch Shipped" Problem

If you use Claude models through Codex CLI's custom provider support, DeepSWE's findings suggest adding explicit requirement enumeration to your AGENTS.md [^1]:

```markdown
## Implementation Requirements

When a task specifies multiple parallel implementations (e.g., sync and async,
CLI and API, read and write):

1. List ALL required implementations before starting
2. Implement each one in sequence
3. Verify each implementation has its own test coverage
4. Do NOT submit until all branches are complete
```

### 4. Use Behavioural Verifiers in Your Own Testing

DeepSWE's dramatic accuracy improvement (1.1 per cent false-negative vs. 23.9 per cent) comes from testing observable behaviour rather than specific implementations [^1]. Apply the same principle to PostToolUse hooks:

```markdown
## Verification Rules (AGENTS.md)

After implementing a feature:
- Test through public API endpoints, not internal method signatures
- Verify observable output, not implementation structure
- Accept any correct solution shape — do not hard-code expected diffs
```

### 5. Monitor Token Efficiency via `rollout_token_budget`

DeepSWE confirms that token consumption is uncorrelated with success [^1]. Set explicit budgets to prevent wasteful exploration:

```toml
# config.toml
[model]
rollout_token_budget = 150000  # Hard ceiling

[model.auto_compact]
token_limit = 80000  # Trigger compaction before budget exhaustion
```

The configurable rollout token budgets introduced in recent Codex CLI releases track usage across agent threads, provide remaining-budget reminders, and abort turns when exhausted [^7].

## The Contamination Problem Is Not Going Away

DeepSWE's anti-contamination design (original tasks, never-merged solutions, shallow clones) represents the current best practice [^1]. But it is inherently limited to a fixed task set. The paper acknowledges this — solutions stay off the public record *for now*, but leaderboard trajectories are published openly, creating a future contamination vector.

The broader industry is responding. SWE-bench Pro has been complemented by dynamic variants (SWE-bench-Live) that continuously refresh tasks [^2]. For Codex CLI users, the practical takeaway is straightforward: public benchmark scores are a rough filter for model selection. Your private evaluation suite is the ground truth.

```mermaid
flowchart LR
    subgraph Public["Public Benchmarks"]
        A[DeepSWE] -->|Model filter| D[Select candidate models]
        B[SWE-bench Verified] -->|Saturated| D
        C[Terminal-Bench 2.1] -->|Harness comparison| D
    end
    subgraph Private["Private Evaluation"]
        D --> E[codex exec against<br/>your test suite]
        E --> F[Profile-specific<br/>pass rates]
        F --> G[Production<br/>model selection]
    end
```

## Limitations Worth Noting

DeepSWE is not a complete picture:

- **Binary scoring** — no partial credit for near-correct solutions [^1]
- **Five languages only** — missing C++, Java, and the enterprise ecosystems [^1]
- **Functional testing only** — no evaluation of code quality, readability, or performance [^1]
- **Single harness** — mini-swe-agent's bash-only interface bypasses vendor-tuned editing primitives [^3]
- **Audit sample size** — judge disagreement rates rest on small event counts within 735 audited rollouts [^1]

These are honest limitations that the authors acknowledge. They do not diminish the benchmark's core contribution: restoring meaningful differentiation between frontier models at a time when the previous yardstick had stopped telling us anything useful.

## Citations

[^1]: Huang, W., Lee, C., Tng, L. & Ge, S. (2026). *DeepSWE: Measuring Frontier Coding Agents on Original, Long-Horizon Engineering Tasks*. arXiv:2607.07946. [https://arxiv.org/abs/2607.07946](https://arxiv.org/abs/2607.07946)

[^2]: "SWE-bench Lost Its Edge, DeepSWE Shows Which Coding AI Actually Works." *Medium / Cogni Down Under*, June 2026. [https://medium.com/@cognidownunder/swe-bench-lost-its-edge-deepswe-shows-which-coding-ai-actually-works-0104376e34cf](https://medium.com/@cognidownunder/swe-bench-lost-its-edge-deepswe-shows-which-coding-ai-actually-works-0104376e34cf)

[^3]: mini-swe-agent GitHub repository. SWE-agent project. [https://github.com/SWE-agent/mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent)

[^4]: "DeepSWE Leaderboard & Scores — July 2026." BenchLM.ai. [https://benchlm.ai/benchmarks/deepSwe](https://benchlm.ai/benchmarks/deepSwe)

[^5]: "Terminal-Bench 2.1 and the June 2026 Benchmark Landscape." *Codex Knowledge Base*, June 2026. [https://codex.danielvaughan.com/2026/06/11/terminal-bench-2-1-june-2026-benchmark-landscape-codex-cli-harness-engineering-model-scores/](https://codex.danielvaughan.com/2026/06/11/terminal-bench-2-1-june-2026-benchmark-landscape-codex-cli-harness-engineering-model-scores/)

[^6]: Codex CLI command-line reference — `codex exec`. OpenAI Developers. [https://developers.openai.com/codex/cli/reference](https://developers.openai.com/codex/cli/reference)

[^7]: Codex CLI Changelog — July 2026. OpenAI Developers. [https://developers.openai.com/codex/changelog](https://developers.openai.com/codex/changelog)
