---
title: "GPT-5.1-Codex-Max: Long-Horizon Tasks, Native Compaction and 24-Hour Sessions"
parent: "Articles"
nav_order: 92
---

![Sketchnote: GPT-5.1-Codex-Max: Long-Horizon Tasks, Native Compaction and 24-Hour Sessions](/sketchnotes/articles/2026-03-29-codex-max-long-horizon-tasks.png)


# GPT-5.1-Codex-Max: Long-Horizon Tasks, Native Compaction and 24-Hour Sessions


Most Codex tasks complete in minutes. But some tasks — a full microservice migration, a large-scale refactor across 50+ files, a days-long debugging session — run into context limits before they finish. GPT-5.1-Codex-Max was built specifically for these long-horizon scenarios.

---

## What Makes GPT-5.1-Codex-Max Different

GPT-5.1-Codex-Max is the first Codex model trained to operate *natively* across multiple context windows. This matters because earlier compaction strategies used external scaffolding (the harness summarising older turns and injecting a summary) — the model wasn't trained to expect those summaries, leading to coherence failures on very long tasks.

With Codex-Max, compaction is trained into the model itself:

- As the session approaches its context limit, the model summarises essential state: variable names, architectural decisions, outstanding bugs, pending steps
- It carries that summary into a fresh context window and continues seamlessly
- This process repeats as many times as needed

**Result:** OpenAI has observed GPT-5.1-Codex-Max working autonomously for **more than 24 hours** on a single task — persistently iterating on implementation, fixing test failures, and ultimately delivering a working result.

---

## When to Use Codex-Max

GPT-5.1-Codex-Max is the right choice when:

- The task is likely to exceed a single context window (large refactors, migrations, multi-file audits)
- You want to fire-and-forget an overnight or weekend-length job
- The task requires sustained coherence across many decision points
- You're working in a regulated environment where you want complete audit continuity

For everyday tasks — fix this bug, write this feature, review this PR — `gpt-5-codex` or `gpt-5.3-codex` are faster and more cost-effective.

---

## Benchmarks: The SWE-Lancer IC SWE Result

GPT-5.1-Codex-Max scores **79.9%** on SWE-Lancer IC SWE — a significant jump from GPT-5.1-Codex's 66.3%.

### What is SWE-Lancer?

SWE-Lancer is an OpenAI benchmark of **1,488 real freelance software engineering tasks** sourced from Upwork and Expensify, worth $1 million USD in real-world payouts. Tasks range from $50 bug fixes to $32,000 feature implementations. End-to-end tests are triple-verified by experienced engineers.

Why it's more meaningful than SWE-Bench Verified: it uses *real economic value* as the scoring unit. A 79.9% result on SWE-Lancer IC SWE means the model can complete the equivalent of ~$798K worth of freelance IC software engineering tasks — compared to 26.2% (Claude 3.5 Sonnet) and 8.6% (GPT-4o) when the benchmark launched in early 2025.

Other benchmark scores (all with compaction + `xhigh` reasoning):

| Benchmark | GPT-5.1-Codex-Max | GPT-5.1-Codex | Notes |
|-----------|-------------------|---------------|-------|
| SWE-Lancer IC SWE | **79.9%** | 66.3% | Real freelance tasks |
| SWE-Bench Verified (n=500) | **77.9%** | 73.7% | Standard synthetic benchmark |
| Terminal-Bench 2.0 (n=89) | **58.1%** | 52.8% | Terminal-native task sequences |

---

## The `xhigh` Reasoning Effort Level

GPT-5.1-Codex-Max introduced a new reasoning effort level: `xhigh`. This is above `high` — the model "thinks for an even longer period" before responding.

```toml
# .codex/config.toml — for long-horizon overnight tasks
model = "gpt-5.1-codex-max"
model_reasoning_effort = "xhigh"
```

Key data point: GPT-5.1-Codex-Max at **`medium` reasoning effort** already outperforms GPT-5.1-Codex at `medium`, while using **30% fewer thinking tokens**. Use `xhigh` for:

- Hard algorithmic problems
- Security audits requiring deep analysis
- Tasks where quality matters more than speed
- Long-horizon runs (the extra thinking pays off over a 24h session)

**Cost note:** `xhigh` carries a significant token multiplier (~8–15× vs `minimal`). For long-horizon tasks, this is usually justified — the model is less likely to go off-track, reducing expensive restarts.

---

## 30% Token Efficiency Gain

GPT-5.1-Codex-Max completes equivalent tasks using roughly **30% fewer tokens** than its predecessor. For long-horizon runs where you're burning credits over hours, this is significant.

Combined with cached input pricing (the model reuses previous context where possible), the effective cost of a 24-hour session can be lower than you'd expect from a frontier model.

---

## Windows Support

GPT-5.1-Codex-Max is the **first Codex model explicitly trained for Windows environments**. If your team includes Windows developers or your CI runs on Windows-hosted runners, this matters for reliability.

---

## Practical Configuration for Long-Horizon Jobs

```toml
# .codex/agents/overnight-worker.toml
name = "overnight-worker"
model = "gpt-5.1-codex-max"
reasoning_effort = "xhigh"
developer_instructions = """
You are a long-horizon agent. Your task may span many context windows.
At each compaction boundary:
1. Summarise the current state in a structured note
2. List: completed steps, current blocker, next 3 steps
3. Preserve all variable names, file paths, and architectural decisions verbatim

Never truncate work. If you hit a context limit, compact and continue.
"""
```

Fire it off via `codex exec`:

```bash
codex exec \
  --agent overnight-worker \
  --dangerously-bypass-approvals-and-sandbox \
  "Migrate the payments service from Stripe v7 to v8 API. All tests must pass. File a PR when complete."
```

Monitor via the Codex app sidebar — the thread will show compaction events as the session progresses.

---

## When NOT to Use Codex-Max

- **Short, bounded tasks**: The model's compaction overhead adds latency on tasks that would finish in a single context window anyway. Use `gpt-5-codex` or `gpt-5.3-codex-spark` for these.
- **Subagent workers**: Use `gpt-5.4-mini` or `gpt-5-codex` for parallel worker subagents in your agentic pod. Reserve Codex-Max for the orchestrator or for standalone long-running jobs. See [Subagent Model Routing: Mini for Workers](../notes/best-practices/#tip-23--subagent-model-routing-mini-for-workers-full-for-orchestrators).
- **Interactive sessions**: If you're steering the agent in real time, shorter context windows keep you in the loop. Codex-Max's strength is *autonomous* long-horizon runs.

---

## Model Lineage Context

GPT-5.1-Codex-Max's place in the lineage:

```
gpt-5-codex (current recommended)
    ↑ gpt-5.3-codex (better tool reliability, Tau2-bench)
        ↑ gpt-5.2-codex (native compaction, context rollup)
            ↑ gpt-5.1-codex-max (24h sessions, SWE-Lancer 79.9%)
```

It has been succeeded by `gpt-5.2-codex` (better native compaction) and `gpt-5.3-codex` (current recommended for most tasks). But for the specific use case of fire-and-forget long-horizon jobs, Codex-Max remains a valid choice — particularly for teams already optimised around its benchmark characteristics.

---

## Citations

- [GPT-5.1-Codex-Max System Card](https://openai.com/index/gpt-5-1-codex-max-system-card/) — safety mitigations, cybersecurity capability assessment
- [SWE-Lancer Benchmark](https://openai.com/index/swe-lancer/) (OpenAI, Feb 2025) — 1,488 Upwork tasks, $1M valuation, arXiv:2502.12115
- [Building more with GPT-5.1-Codex-Max](https://openai.com/index/gpt-5-1-codex-max/) — official announcement with benchmark data
- [VentureBeat: 24-hour task evaluation](https://venturebeat.com/ai/openai-debuts-gpt-5-1-codex-max-coding-model-and-it-already-completed-a-24) — reporting on internal evaluation sessions
