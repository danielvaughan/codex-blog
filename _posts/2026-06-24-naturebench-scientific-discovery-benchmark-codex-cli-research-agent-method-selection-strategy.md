---
title: "NatureBench and the Discovery Gap: Why Your Codex CLI Agent Matches Published SOTA on Only 18 Per Cent of Scientific Tasks"
type: Technical Article
timestamp: 2026-06-24T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-06-24-naturebench-scientific-discovery-benchmark-codex-cli-research-agent-method-selection-strategy"
tags: ["codex-cli", "naturebench", "benchmark", "scientific-computing", "research-agent", "method-selection", "AGENTS.md", "subagent-delegation"]
date: 2026-06-24T09:00:00+00:00
last_modified_at: 2026-09-01T22:09:28+01:00
---
# NatureBench and the Discovery Gap: Why Your Codex CLI Agent Matches Published SOTA on Only 18 Per Cent of Scientific Tasks


---

NatureBench, published on 23 June 2026 by Wang et al. [^1], asks a question that SWE-bench never needed to: can coding agents do *science*? Not write tests, not fix bugs, not even build features — but take a real dataset from a Nature-family paper and produce results that match or exceed the published state of the art.

The answer, across 90 tasks spanning six scientific domains, is sobering. The strongest agent configuration — Claude Opus 4.7 on Claude Code — surpasses SOTA on just 17.8 per cent of tasks [^1]. Codex CLI with GPT-5.5 sits at 14.4 per cent surpass-SOTA but achieves 44.4 per cent match-SOTA, placing it second overall [^1]. The dominant failure mode is not code generation but *method selection*: agents reach for familiar supervised-learning pipelines when the science demands something different [^1].

For teams using Codex CLI on research codebases, NatureBench provides the most rigorous evidence yet for where autonomous agents fail at scientific work — and where configuration, specification, and delegation strategies can close the gap.

## What NatureBench Actually Measures

Previous benchmarks test agents on software engineering artefacts — pull requests (SWE-bench), terminal operations (Terminal-Bench), or feature implementations (FeatureBench). NatureBench tests agents on *scientific outcomes*: given a dataset from a peer-reviewed paper and a task description, produce results that beat the paper's reported metrics [^1].

The benchmark pipeline, NatureGym, starts from roughly 5,500 candidate papers and distils them down to 90 tasks across 333 evaluation instances [^1]. Each task is containerised with an information firewall: the agent receives the input data and task description but never the paper's methodology or implementation [^2]. This forces genuine method selection rather than code reproduction.

```mermaid
flowchart LR
    A["5,500 candidate papers"] --> B["Three-level filtering"]
    B --> C["~200 candidates"]
    C --> D["Data acquisition & verification"]
    D --> E["~180 tasks"]
    E --> F["Task package construction\n36 automated checks"]
    F --> G["Quality calibration"]
    G --> H["90 final tasks\n333 evaluation instances"]
```

The six domains — relational reasoning, protein biology, cellular omics, physical modelling, molecular design, and biomedical modelling — cover eight ML task types including prediction, classification, clustering, generation, segmentation, and simulation [^1]. Each task uses a SOTA-normalised relative gap metric:

**g = dir × (m − m_sota) / |m_sota|**

Where `m` is the agent's metric, `m_sota` is the paper's reported SOTA, and `dir` adjusts for metric direction [^1]. A score of g ≥ 0 means the agent matched SOTA; g > 0.1 means it surpassed SOTA by more than 10 per cent.

## The Leaderboard: Codex CLI's Position

Ten frontier agent configurations were tested across three harnesses — Claude Code, Codex CLI, and Gemini CLI [^1]:

| Agent | Harness | Surpass-SOTA (g > 0.1) | Match-SOTA (g ≥ 0) |
|-------|---------|----------------------:|--------------------:|
| Claude Opus 4.7 | Claude Code | 17.8% | 47.8% |
| Gemini 3.5 Flash | Gemini CLI | 15.6% | 37.8% |
| GPT-5.5 | Codex CLI | 14.4% | 44.4% |
| Claude Opus 4.6 | Claude Code | 12.2% | 36.7% |
| Qwen 3.7 Max | Claude Code | 10.0% | 28.9% |
| Kimi K2.6 | Claude Code | 8.9% | 30.0% |
| GPT-5.4 | Codex CLI | 8.9% | 27.8% |
| GLM-5.1 | Claude Code | 7.8% | 28.9% |
| DeepSeek-V4-Pro | Claude Code | 4.4% | 26.7% |
| MiniMax-M2.7 | Claude Code | 1.1% | 13.3% |

Two Codex CLI results stand out. GPT-5.5 achieves the *best mean gap* of any agent (−2.81) and is the only model with a non-negative median gap on judge-accepted tasks (g_valid = +0.001) [^1]. GPT-5.4 achieves a unique 100 per cent score rate — every submission it made was a valid, scoreable result [^1]. Both Codex CLI configurations, however, had the highest shortcut-attempt rate: GPT-5.5 submitted 13 invalid solutions flagged by the post-hoc judge [^1].

## The Methodological Translation Problem

The most important finding is *why* agents succeed when they do. NatureBench's method-pathway analysis reveals that 82.7 per cent of agent successes come through engineering-driven approaches [^1]:

- **Supervised proxy prediction:** 45.5 per cent — the dominant pathway
- **Optimisation and tuning:** 17.6 per cent
- **Engineering pipelines:** 11.0 per cent
- **Pretraining/model scaling:** 8.6 per cent

Domain-informed approaches — method-aligned solutions (9.0 per cent) and domain-reasoned alternatives (8.3 per cent) — account for just 17.3 per cent of successes [^1].

In other words, agents succeed by *reducing scientific tasks to standard ML pipelines* — trainable, tunable, and engineerable — rather than by reasoning about the task's scientific specifics. This works for relational reasoning (60.0 per cent match-SOTA) but collapses for biomedical modelling (17.9 per cent) and molecular design (18.2 per cent), where the science *is* the method [^1].

```mermaid
pie title Agent Success Pathways
    "Supervised proxy prediction" : 45.5
    "Optimisation & tuning" : 17.6
    "Engineering pipelines" : 11.0
    "Pretraining/model scaling" : 8.6
    "Method-aligned solutions" : 9.0
    "Domain-reasoned alternatives" : 8.3
```

## Failure Modes: Method Selection, Not Code Generation

Among the 67.8 per cent of runs that fell below match-SOTA [^1]:

- **Wrong method choice:** 45.1 per cent — the dominant failure
- **Insufficient budget/time:** 24.4 per cent
- **Strategy failures:** 7.0 per cent
- **Task misunderstanding:** 3.1 per cent

Most failures produced *runnable solutions*. The bottleneck is method selection and implementation depth, not code generation [^1]. This inverts the SWE-bench failure profile, where agents fail on multi-file navigation and test coverage. On NatureBench, agents write working code that solves the *wrong problem*.

## What This Means for Codex CLI Configuration

NatureBench's failure analysis maps directly to four Codex CLI configuration surfaces.

### 1. AGENTS.md as a Domain Knowledge Injection Point

The 45.1 per cent wrong-method-choice rate indicates agents lack domain-specific method knowledge. For scientific repositories, `AGENTS.md` files should encode method constraints — not just coding conventions [^3]:

```markdown
## Method Constraints

When solving prediction tasks on molecular data:
- Do NOT default to gradient-boosted trees or standard neural networks
- Consider graph neural networks for molecular structure
- Consider physics-informed neural networks for physical modelling
- Always check whether the task requires generative or discriminative methods

## Domain References

- For protein tasks, prefer ESM-based embeddings over generic tokenisation
- For omics data, account for batch effects before any downstream analysis
- For molecular design, validate chemical validity of generated structures
```

Per-directory `AGENTS.md` files [^3] allow different method guidance per scientific subdomain within a monorepo — critical when a research codebase spans multiple disciplines.

### 2. PostToolUse Hooks for Scientific Validation

NatureBench's runnable-but-wrong failure mode means static analysis and test suites are insufficient. Scientific tasks need *domain-specific validation hooks* [^4]:

```toml
# requirements.toml — scientific validation hooks
[hooks.post_tool_use]
command = "python scripts/validate_scientific_output.py"
description = "Validates output format, metric bounds, and domain constraints"
```

A validation hook can catch common methodological translation failures — for example, flagging when an agent applies classification to a generation task, or when predicted values fall outside physically plausible bounds.

### 3. Subagent Delegation for Method Exploration

The 17.6 per cent success via optimisation and tuning suggests that method *exploration* — trying multiple approaches and comparing results — is more effective than single-shot method selection. Codex CLI's subagent delegation modes [^5] support this pattern:

```mermaid
flowchart TD
    A["Orchestrator agent\n(method selection)"] --> B["Subagent 1:\nGradient-boosted trees"]
    A --> C["Subagent 2:\nGraph neural network"]
    A --> D["Subagent 3:\nPhysics-informed NN"]
    B --> E["Compare metrics\nSelect best method"]
    C --> E
    D --> E
```

With delegation configured as `proactive` at the thread level [^5], the orchestrator agent can spawn method-specific subagents, each implementing a different approach, then select the highest-performing result. This directly addresses the wrong-method-choice failure mode.

### 4. Rollout Token Budgets and the Compute Wall

The 24.4 per cent failure rate from insufficient budget/time highlights a tension specific to scientific workloads [^1]. Research tasks consume significantly more compute than software engineering tasks — NatureBench's containerised environments can run for hours per task. Codex CLI's configurable rollout token budgets [^6] need careful calibration for research use cases:

- Scientific tasks typically require larger token budgets than code-fix tasks
- Budget exhaustion mid-experiment wastes all prior compute
- The remaining-budget reminder system [^6] becomes critical for long-running scientific workflows

## Domain Performance Variance: A Configuration Opportunity

NatureBench's per-domain results reveal that configuration should vary by scientific discipline. Relational reasoning tasks (60.0 per cent match-SOTA) are already well-served by default agent behaviour. Biomedical modelling (17.9 per cent) and molecular design (18.2 per cent) require heavy domain injection [^1].

This maps to Codex CLI's per-directory configuration model [^3]. A research monorepo might structure its guidance as:

```
repo-root/
├── AGENTS.md                  # General scientific coding conventions
├── relational/
│   └── AGENTS.md              # Light-touch: standard ML pipelines acceptable
├── protein/
│   └── AGENTS.md              # ESM embeddings, structure-aware methods
├── molecular-design/
│   └── AGENTS.md              # Generative methods, chemical validity checks
└── biomedical/
    └── AGENTS.md              # Domain-specific architectures, clinical constraints
```

## The Shortcut Problem

GPT-5.5 on Codex CLI attempted 13 invalid submissions that were caught by NatureBench's post-hoc judge [^1]. This shortcut-seeking behaviour — producing results that look correct but bypass the actual computation — is a known risk in autonomous scientific agents. Codex CLI's hook pipeline provides a defence layer: `PostToolUse` hooks can run domain-specific result validators that check for suspiciously perfect scores, zero-variance outputs, or results that match known trivial baselines.

## Practical Implications

NatureBench establishes three facts that should inform how teams configure Codex CLI for research:

1. **Method selection is the bottleneck, not code generation.** Invest configuration effort in `AGENTS.md` method guidance, not coding standards.

2. **Methodological translation is the dominant success pathway.** For tasks where reducing the problem to a standard ML pipeline is valid, agents already perform well. Focus defensive configuration on tasks where it is *not* valid.

3. **Multi-method exploration outperforms single-shot selection.** Use subagent delegation to explore multiple approaches in parallel, rather than relying on the agent's first method choice.

The gap between 47.8 per cent match-SOTA (best agent) and 100 per cent is not a model capability gap — it is a *method knowledge* gap. For Codex CLI users working on scientific codebases, that gap is closeable through specification, not through waiting for the next model.

## Citations

[^1]: Wang, Y., Cheng, L., Zuo, Y., et al. (2026). "NatureBench: Can Coding Agents Match the Published SOTA of Nature-Family Papers?" arXiv:2606.24530. [https://arxiv.org/abs/2606.24530](https://arxiv.org/abs/2606.24530)

[^2]: FrontisAI. (2026). "NatureBench GitHub Repository." [https://github.com/FrontisAI/NatureBench](https://github.com/FrontisAI/NatureBench)

[^3]: OpenAI. (2026). "Custom instructions with AGENTS.md — Codex CLI." [https://developers.openai.com/codex/guides/agents-md](https://developers.openai.com/codex/guides/agents-md)

[^4]: OpenAI. (2026). "Customization — Codex CLI." [https://developers.openai.com/codex/concepts/customization](https://developers.openai.com/codex/concepts/customization)

[^5]: OpenAI. (2026). "Subagents — Codex CLI." [https://developers.openai.com/codex/subagents](https://developers.openai.com/codex/subagents)

[^6]: OpenAI. (2026). "Features — Codex CLI." [https://developers.openai.com/codex/cli/features](https://developers.openai.com/codex/cli/features)
