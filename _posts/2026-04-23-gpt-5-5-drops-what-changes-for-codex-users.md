---
title: "GPT-5.5 Drops: What Changes for Codex Users"
description: "Six weeks. That is the gap between GPT-5.4 and GPT-5.5. OpenAI released its newest frontier model on 23 April 2026, rolling it out simultaneously to ChatGPT."
date: 2026-04-23T19:30:00+00:00
last_modified_at: 2026-06-13T20:24:13+01:00
tags:
  - gpt-5.5
  - model-upgrade
  - benchmarks
  - pricing
  - codex
  - terminal-bench
  - swe-bench
  - hallucination
  - token-efficiency
  - competitive-landscape
---

![Sketchnote diagram for: GPT-5.5 Drops: What Changes for Codex Users](/sketchnotes/articles/2026-04-23-gpt-5-5-drops-what-changes-for-codex-users.png)


# GPT-5.5 Drops: What Changes for Codex Users

Six weeks. That is the gap between GPT-5.4 and GPT-5.5. OpenAI released its newest frontier model on 23 April 2026, rolling it out simultaneously to ChatGPT and Codex for all paid tiers — Plus, Pro, Business, and Enterprise[^1]. The internal codename is "Spud"[^2], and it arrives at a moment when the competitive pressure from Anthropic's Claude Mythos Preview has been intensifying.

This is what the upgrade means for the 4 million active Codex users.

## The numbers

The headline benchmark is Terminal-Bench 2.0, which tests complex command-line workflows requiring planning, iteration, and tool coordination. GPT-5.5 scores **82.7%** — a 7-point jump over GPT-5.4 and a new state-of-the-art result that narrowly beats Anthropic's Claude Mythos Preview on the same benchmark[^3].

| Benchmark | GPT-5.4 | GPT-5.5 | Change |
|-----------|---------|---------|--------|
| Terminal-Bench 2.0 | ~75% | **82.7%** | +7pp (SOTA) |
| SWE-bench Pro | 57.7% | **58.6%** | +0.9pp |
| SWE-bench | — | **88.7%** | — |
| GDPval (44 occupations) | — | **84.9%** | New |
| MMLU | — | **92.4%** | — |

The SWE-bench Pro improvement is modest — less than one percentage point. But Terminal-Bench 2.0 is where Codex CLI users live: multi-step terminal tasks with real tool usage, file edits, and iterative problem-solving. A 7-point gain there translates directly to more tasks completing successfully on the first pass.

GDPval is a newer benchmark testing agents across 44 different occupations — not just coding. The 84.9% score reflects OpenAI's push toward general agentic capability. Greg Brockman described GPT-5.5 as "a big step towards more agentic and intuitive computing" — a model that can "look at an unclear problem and figure out what needs to happen next" with minimal guidance[^1].

## The hallucination drop matters more than the benchmarks

For enterprise users, the most significant number may be the **60% reduction in hallucinations** compared to previous versions[^4]. Bank of New York CIO Leigh-Ann Russell highlighted "response quality — but also a really impressive hallucination resistance" as "critical" for regulated financial institutions[^1].

In Codex CLI terms, hallucination reduction means fewer fabricated file paths, fewer invented API endpoints, fewer test assertions that check the wrong thing. It means the verification burden on the Quality Engineer in an agentic pod decreases — not to zero, but measurably.

## Fewer tokens, better results

GPT-5.5 is not just more capable — it is more token-efficient. OpenAI reports that it "delivers better results with fewer tokens than GPT-5.4 for most users"[^5]. For Codex, the team "carefully tuned the experience" to optimise this[^5].

This matters because the API pricing doubled:

| | GPT-5.4 | GPT-5.5 | Change |
|--|---------|---------|--------|
| Input | \$2.50/M tokens | \$5.00/M tokens | 2× |
| Output | \$15.00/M tokens | \$30.00/M tokens | 2× |

The sticker price is twice as high. But if the model solves the problem in one pass instead of two, or in 800 output tokens instead of 1,500, the effective cost per task may stay roughly the same — or even drop. The token efficiency claim is critical to the pricing story and needs real-world validation in the coming days.

A new **Fast mode** generates tokens 1.5× faster at 2.5× the cost — useful for latency-sensitive workflows where the developer is waiting in the terminal.

## The 400K context window in Codex

GPT-5.5 brings a **400K token context window** in Codex, with up to 12M tokens available in other implementations[^5]. For Codex CLI users, the practical impact depends on how the context is managed — a 400K window is large enough to hold a substantial portion of most codebases, but effective context engineering (CLAUDE.md equivalents, targeted file selection, specification-driven prompts) remains essential.

The larger context window is most valuable for background agents working on complex, multi-file tasks where the full dependency graph needs to be visible. For foreground interactive sessions, context discipline still matters more than window size.

## Competitive positioning

The timing is not accidental. The release "addresses narratives suggesting OpenAI lost momentum against Anthropic, particularly in enterprise competition"[^1]. The VentureBeat headline captures the dynamic: GPT-5.5 "narrowly beats Anthropic's Claude Mythos Preview on Terminal-Bench 2.0"[^3].

The agentic coding landscape as of today:

| Tool | Primary Model | Terminal-Bench 2.0 | SWE-bench Pro |
|------|--------------|-------------------|---------------|
| **Codex CLI** | GPT-5.5 | **82.7%** | 58.6% |
| **Claude Code** | Claude Mythos Preview | ~82% | — |
| **Cursor** | Multi-model (GPT-5.5, Claude) | — | — |
| **GitHub Copilot** | GPT-5.3-Codex (LTS) / multi | — | — |

The race is close. The gap between GPT-5.5 and Claude Mythos Preview on Terminal-Bench 2.0 is narrow. Anthropic will respond. The practical difference for users choosing between tools is now less about model capability and more about the agent harness: approval modes, sandbox security, context engineering, orchestration patterns, and enterprise governance.

## What a mathematics professor proved in 11 minutes

Fortune reported that a mathematics professor used GPT-5.5 with Codex to build an algebraic geometry application from a single prompt in 11 minutes[^1]. The anecdote matters less for the specific use case and more for what it demonstrates: the combination of stronger reasoning, reduced hallucination, and better token efficiency means that the practical ceiling for single-prompt tasks has risen noticeably.

For agentic pod workflows — where the Value Engineer writes a specification and the agent executes — this translates to fewer iteration cycles, less rework, and faster time from specification to working code.

## NVIDIA's 10,000-person enterprise validation

NVIDIA published a blog post on launch day revealing the most concrete enterprise Codex adoption data yet[^6]. Over **10,000 NVIDIA employees** got early access to Codex + GPT-5.5 — not just engineers, but legal, marketing, finance, sales, HR, and operations teams. Engineers described results as "mind-blowing" and "life-changing."

The infrastructure numbers are equally striking: GPT-5.5 runs on NVIDIA's GB200 NVL72 rack-scale systems, which deliver **35× lower cost per million tokens** and **50× higher token throughput per megawatt** compared to prior hardware generations. A 100,000-GPU cluster completed multiple frontier-scale training runs with new reliability benchmarks.

For the agentic pod thesis, the NVIDIA case study is significant: it demonstrates that Codex adoption expands beyond engineering once model quality crosses a reliability threshold. When legal and finance teams adopt coding agents, the organisational impact is fundamentally different from developer-only tooling.

## What to do right now

**If you are on a subscription tier (Plus/Pro/Business/Enterprise):** GPT-5.5 is already available. Your Codex sessions should be using it now. Check your model setting.

**If you are on API key billing:** The 2× price increase is real. Run a few representative tasks and compare token usage against GPT-5.4. If the efficiency gains hold — fewer tokens per task — the effective cost increase may be manageable. If not, GPT-5.4 remains available and is now the better value option for routine tasks.

**If you are running an agentic pod at enterprise scale:** The hallucination reduction is the most operationally significant change. Update your AGENTS.md and verification workflows to take advantage of the higher baseline reliability, but do not reduce verification gates yet. Let the 60% claim be validated by your own metrics before loosening controls.

**For the book:** Four chapters need updates: ch03 (what's new), ch05 (competing tools), ch11 (model selection), ch17 (cost management).

---

## Citations

[^1]: Levy, R. (2026, April 23). "OpenAI launches GPT-5.5 just weeks after GPT-5.4 as AI race accelerates." *Fortune*. <https://fortune.com/2026/04/23/openai-releases-gpt-5-5/>

[^2]: Axios. (2026, April 23). "OpenAI releases 'Spud' GPT-5.5 model." <https://www.axios.com/2026/04/23/openai-releases-spud-gpt-model>

[^3]: VentureBeat. (2026, April 23). "OpenAI's GPT-5.5 is here, and it's no potato: narrowly beats Anthropic's Claude Mythos Preview on Terminal-Bench 2.0." <https://venturebeat.com/technology/openais-gpt-5-5-is-here-and-its-no-potato-narrowly-beats-anthropics-claude-mythos-preview-on-terminal-bench-2-0>

[^4]: StartupFortune. (2026, April 23). "OpenAI's GPT-5.5 benchmarks show a 60% hallucination drop and coding skills that rival senior engineers." <https://startupfortune.com/openais-gpt-55-benchmarks-show-a-60-hallucination-drop-and-coding-skills-that-rival-senior-engineers/>

[^5]: 9to5Mac. (2026, April 23). "OpenAI upgrades ChatGPT and Codex with GPT-5.5: 'a new class of intelligence for real work'." <https://9to5mac.com/2026/04/23/openai-upgrades-chatgpt-and-codex-with-gpt-5-5-a-new-class-of-intelligence-for-real-work/>

[^6]: NVIDIA Blog. (2026, April 23). "OpenAI's New GPT-5.5 Powers Codex on NVIDIA Infrastructure — and NVIDIA Is Already Putting It to Work." <https://blogs.nvidia.com/blog/openai-codex-gpt-5-5-ai-agents/>
