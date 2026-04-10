---
title: "Evaluating Codex Agents: Evals, Long-Horizon Benchmarks, and the 4-File Pattern"
date: 2026-03-28
description: "How to evaluate whether your Codex agent actually did the right thing — from quick skill evals to 25-hour autonomous runs."
tags:
  - workflow-patterns
  - testing
  - evals
  - evaluation
  - long-horizon
  - longcli-bench
  - codex-exec
  - codex-cli
---

![Sketchnote diagram for: Evaluating Codex Agents: Evals, Long-Horizon Benchmarks, and the 4-File Pattern](/sketchnotes/articles/2026-03-28-evaluating-codex-agents-evals-longhorizon.png)

*Research compiled 2026-03-28. Sources: [OpenAI eval-skills guide](https://developers.openai.com/blog/eval-skills), [Run long horizon tasks with Codex](https://developers.openai.com/blog/run-long-horizon-tasks-with-codex) (Feb 23, 2026), [LongCLI-Bench](https://arxiv.org/abs/2602.14337) (Feb 2026).*

---

Agentic coding breaks the assumptions that make unit tests sufficient. When Codex runs for hours or spawns a dozen subagents, "did the tests pass?" is necessary but not enough — it doesn't tell you whether the agent took the right path, consumed a reasonable number of tokens, or avoided modifying files it shouldn't have touched.

This article covers three layers of Codex evaluation: skill-level evals for short tasks, the 4-file durable memory pattern for long-horizon runs, and what LongCLI-Bench reveals about the state of the art.

---

## Layer 1: Skill Evals (Short Tasks, ~10–20 Prompts)

OpenAI's [eval-skills guide](https://developers.openai.com/blog/eval-skills) defines evaluation for Codex agents as: *a prompt → a captured run (trace + artifacts) → a set of checks → a comparable score*.

**Three goal types to measure:**

| Goal type | Question | How to check |
|-----------|----------|-------------|
| **Process** | Did Codex invoke the right tools in the right order? | JSONL trace: check `command_execution` events |
| **Style** | Does output follow project conventions? | `--output-schema` rubric grading |
| **Efficiency** | Did it get there without thrashing? | Token count + step count thresholds |

**The core eval loop:**

```bash
# 1. Run skill and capture trace
codex exec --json "Run the release notes skill" > trace.jsonl

# 2. Deterministic checks: did it call expected tools?
cat trace.jsonl | jq '.events[] | select(.type == "command_execution") | .command'

# 3. Structured rubric grading
codex exec --output-schema release-eval-schema.json "Grade this output: $(cat output.md)"
```

**Scale guidance:** 10–20 prompts is enough to surface regressions in a single skill. Grow the dataset as you encounter real failures. For CI, use an 80–90% gate: `N out of M eval samples must pass`.

**Measure variance:** The same prompt can produce different results. Run with `--repeat 3` and compare:

```bash
codex exec --repeat 3 "Run the security audit skill" 2>&1 | grep "pass_rate"
```

**Cost guardrails:** Set thresholds to catch regressions:
- A security audit skill should cost < $0.25 and complete in < 30 seconds
- If latency or cost spikes, the agent is thrashing

---

## Layer 2: Long-Horizon Run Patterns (Hours, Not Minutes)

The [OpenAI blog post](https://developers.openai.com/blog/run-long-horizon-tasks-with-codex) (Feb 23, 2026) documents a GPT-5.3-Codex run of **~25 hours, 13M tokens, ~30K lines of code** to build a design tool from scratch. The key enabler wasn't the model — it was the **4-file durable project memory pattern**.

### The 4-File Pattern

Instead of relying on context window alone, maintain four markdown files as external state:

| File | Purpose |
|------|---------|
| `Prompt.md` | Frozen spec — non-negotiable requirements, never modified during the run |
| `Plan.md` | Verifiable milestones with acceptance criteria (checked off as work proceeds) |
| `Implement.md` | Operational instructions, validation protocols, coding conventions |
| `Documentation.md` | Real-time status and decision log — Codex updates this as it works |

**Why it works:** Context compaction prunes conversation history, but these files survive. When Codex compacts and re-reads them, it regains orientation without losing project state. Prompt.md acts as a constraint anchor — the model can't drift from the original spec.

**AGENTS.md integration:**

```markdown
## Long Horizon Run Protocols

When given a multi-day task:
1. Read Prompt.md — do not modify
2. Read Plan.md — update status as milestones complete
3. Read Documentation.md — append decisions and blockers
4. After each milestone: run verification commands (lint, typecheck, tests, build)
5. Repair failures before continuing to next milestone
```

**Practical verification rhythm:** After each milestone, Codex ran lint checks, typechecks, the test suite, and a build. Failures were repaired before continuing. This continuous verification prevented error accumulation over the long run.

---

## Layer 3: LongCLI-Bench — What Research Says

**LongCLI-Bench** ([arXiv:2602.14337](https://arxiv.org/abs/2602.14337), submitted Feb 15, 2026) is the first benchmark designed specifically to test long-horizon CLI agent behaviour at realistic task lengths.

**Key findings:**

- All tested agents — including Claude Code (Claude-Opus-4.6) and Codex (GPT-5.3-Codex) — achieve **< 20% pass rate**
- Most failures occur in the **early stage** of tasks (< 30% completion), not at the finish line
- **Self-correction provides only marginal improvement** — the bottleneck is initial planning, not in-flight correction
- Human-agent collaboration (plan injection, interactive guidance) **significantly outperforms fully autonomous runs**

**Benchmark design:**
- 20 curated tasks across 4 engineering categories: from-scratch development, feature addition, bug fixing, refactoring
- Dual-set testing: requirement fulfillment (fail-to-pass) + regression avoidance (pass-to-pass)
- Expert completion time average: **1,000+ minutes** per task (vs 207 min for Terminal-Bench@2)
- Step-level scoring gives granular failure analysis

**Implication for Daniel's agentic pod design:** The < 20% autonomous pass rate reinforces why plan injection (the 4-file pattern) and human milestone checkpoints produce better outcomes than pure "fire-and-forget" automation. The data supports a **supervised autonomy** model: agent executes, human reviews milestones, agent continues.

**Practical rule from LongCLI-Bench:** Front-load planning. Failures cluster at the start of tasks, not the end. A well-structured `Plan.md` with clear acceptance criteria dramatically reduces early-stage failures.

---

## Putting It Together: The Evaluation Stack

```
Short tasks (< 30 min)
  → Skill evals: 10-20 prompts, JSONL trace checks, --output-schema rubric
  → CI gate: 80% pass rate, cost < $0.25, latency < 30s

Medium tasks (30 min – 4 hours)
  → TDD feedback loop: tests define "done", agent iterates until green
  → PostToolUse hook: auto-run tests after every file write
  → /fork to explore alternatives without losing main progress

Long tasks (4+ hours)
  → 4-file durable memory: Prompt.md + Plan.md + Implement.md + Documentation.md
  → Milestone verification: lint + typecheck + tests + build after each milestone
  → gpt-5.1-codex-max for compaction-aware extended runs
  → xhigh reasoning effort for tasks that genuinely resist solution at lower budgets
  → Human checkpoint at each milestone (not continuous monitoring)
```

---

## Model Selection for Evaluation Tasks

| Scenario | Model | Reasoning | Why |
|----------|-------|-----------|-----|
| Eval runs (fast, many repeats) | `gpt-5-codex-mini` | `low` | 4x more included usage, 2x faster |
| Standard task evaluation | `gpt-5-codex` | `medium` | Default; good quality/cost balance |
| Long-horizon autonomous run | `gpt-5.1-codex-max` | `high` | Compaction-aware; designed for extended runs |
| Hard algorithmic / security eval | `gpt-5.3-codex` | `xhigh` | For tasks that resist solution at lower budgets |

---

## Key Takeaway

Evaluation for Codex agents is a spectrum. At the short end, JSONL traces + rubric scoring + 10-prompt eval sets catch regressions in skills cheaply. At the long end, the 4-file durable memory pattern + milestone verification enables coherent 25-hour runs. The benchmark research confirms that neither end is "solved" — but structured context management is what separates 10-minute demos from production-grade agentic workflows.

*Related: [notes/best-practices.md](/codex-resources/notes/best-practices/) | [articles/2026-03-28-test-first-development-codex-tdd-feedback-loop.md](/codex-resources/articles/2026-03-28-test-first-development-codex-tdd-feedback-loop/) | [articles/2026-03-28-codex-agent-loop-deep-dive.md](/codex-resources/articles/2026-03-28-codex-agent-loop-deep-dive/)*
