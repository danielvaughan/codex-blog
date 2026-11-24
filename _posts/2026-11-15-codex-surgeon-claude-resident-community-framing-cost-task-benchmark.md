---
title: '"Codex Surgeon, Claude Resident": How the Developer Community Settled Its Most-Debated Question'
date: 2026-11-15T10:00:00+00:00
last_modified_at: 2026-11-24T17:05:09Z
parent: "Articles"
nav_order: 1235
tags:
  - codex-cli
  - claude-code
  - community
  - benchmarks
  - cost
  - comparison
  - swe-bench
  - terminal-bench
  - sentiment
  - hybrid-workflow
copy_edited: true
rating: 4.5
depth: 4
originality: 5
accuracy: 4
substack: 5
---

The comparison keeps coming up. Codex CLI versus Claude Code. Fast versus careful. Terminal-native versus plan-faithful. Every thread in r/vibecoding, r/ChatGPTCoding, and the GitHub issue tracker eventually circles back to it. The community has been trying to find the right frame for months.

On 15 November 2026, u/Reza__ gave them one.

> "Codex is like a Surgeon and Claude is more like a Surgical Resident."

The post spread fast. By the end of the day it had been quoted in at least four separate threads. It stuck because it named something developers had been experiencing but struggling to articulate. It is not that one tool is better. It is that they are built for different kinds of work at different points in a project.

## What the Data Actually Shows

The analogy is not just metaphor. It maps onto measurable differences.

Morphllm's "15 AI Coding Agents Tested" benchmark tested both tools across SWE-bench Verified — the standard repository-level bug-fixing evaluation — and Terminal-Bench 2.0, which measures performance on terminal-native agentic tasks.

| Tool | SWE-bench Verified | Terminal-Bench 2.0 | Throughput |
|---|---|---|---|
| **Claude Code** | 80.9% (1st) | 74.1% | ~96 tok/s |
| **Codex CLI** | 75.2% (3rd) | 77.3% (1st) | ~240 tok/s |

Claude Code leads on SWE-bench by 5.7 percentage points. Codex CLI leads on Terminal-Bench by 3.2 points and runs at more than twice the throughput. The pattern holds up under cost analysis too: community-reported task costs averaged \$4.43 per task for Codex and \$11.84 for Claude Code — a 2.7× gap.

The surgeon analogy captures exactly this. A surgeon is fast, precise, and expensive per hour but completes a high volume of discrete procedures efficiently. A resident is thorough, deliberate, and deeply engaged with reasoning through the full clinical picture before acting. Both are essential. Neither is a substitute for the other.

## What Codex Does Well

The morphllm assessment identified Codex CLI's specific strengths directly:

- **Throughput**: 240+ tokens per second — roughly 2.5× faster than Opus-class models
- **Terminal-native tasks**: Highest Terminal-Bench 2.0 score of any tool tested (77.3%)
- **Open-source architecture**: Apache 2.0, fully extensible Rust codebase
- **Code review depth**: Flagging logical errors, race conditions, and edge cases at speed

These strengths cluster around a particular workflow pattern: fast, iterative, DevOps-heavy work where you want an agent completing discrete tasks at volume without pausing to explain its reasoning at each step. Codex wrote the code, ran the tests, and pushed the commit before Claude Code finished its plan.

## What Claude Code Does Well

The same assessment is equally clear about where Claude Code leads:

- **Reasoning depth**: Better on complex refactors, architectural decisions, and subtle bug classes
- **Plan-following fidelity**: More conservative execution model; less likely to interpret ambiguous instructions liberally
- **SWE-bench performance**: 80.9% — strongest result in the benchmark for standard repository tasks

This is the resident's strength. When the work requires holding the full architecture in mind across a long session and reasoning carefully about consequences before acting, Claude Code's slower, more deliberate approach is not a disadvantage — it is the point.

## The Flip Side the Analogy Also Explains

The surgeon frame also names the most persistent complaint about Codex CLI. Issue #59740, which accumulated reports from at least five separate users before an automated bot flagged four duplicates, documents a recurring failure mode: Codex loses track of previously agreed decisions during long sessions, re-enters planning loops, and fails to make meaningful progress.

One reporter, after five sessions with GPT-6.1 Sol without meaningful output, explicitly cancelled their €20/month subscription and switched to Claude Code. The post attracted no response from OpenAI.

A surgeon operating outside their specialty is not safer for being fast. The instruction-adherence failure pattern is the domain boundary showing.

A related Linux regression compounds this: on Linux, Codex's context summarisation pass disproportionately weights the most recent turn, causing earlier prompt boundaries — including agreed constraints and task parameters — to drop from context mid-session. The result looks like the agent "forgetting" what it agreed to do an hour ago.

## The Hybrid Workflow That Is Emerging

The community has not settled on a winner. It has settled on a workflow.

The pattern emerging across threads is a division of labour that maps directly onto the analogy:

- **Codex CLI** for fast iteration, file operations, test execution, CI tasks, shell-heavy work, and anything where throughput matters more than reasoning depth
- **Claude Code** for architectural decisions, complex refactors, long planning sessions, and any task where you want the agent to reason carefully before it touches anything

One user described it as "Codex for shipping, Claude for thinking." Another framed it as running Codex on the sprint and Claude Code on the architecture review. The \$4.43/\$11.84 cost split makes the economics of this hybrid approach tractable: use the cheaper, faster tool for volume work and the deeper tool selectively.

## Why the Analogy Will Last

The surgeon/resident frame has persistence because it is structurally accurate rather than tribally convenient. It does not declare a winner. It maps the tools onto a workflow and lets developers make rational decisions about which to reach for.

The morphllm benchmark data backs it up. The cost data backs it up. Even the failure modes back it up — each tool's weaknesses are the shadow of its strengths, legible once you hold the analogy in mind.

Codex CLI's Apache 2.0 licence, 240 token/second throughput, and terminal-native benchmark performance make it the right instrument for surgical iteration at scale. Claude Code's reasoning depth, SWE-bench lead, and plan-following conservatism make it the right instrument for the cases where you need the resident to think before the surgery begins.

The debate was never going to resolve in favour of one tool eliminating the other. The community found a better resolution: a job description for each.

---

*Source material: morphllm.com "15 AI Coding Agents Tested" (April 2026); r/vibecoding u/Reza__ (November 2026); GitHub Issues #59740, #59094, #59269, #58694; community cost benchmarks from r/ChatGPTCoding.*
