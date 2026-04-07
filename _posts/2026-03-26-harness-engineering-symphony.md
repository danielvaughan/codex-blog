---
title: "From Harness Engineering to Symphony: The Autonomous Development Stack"
layout: single
date: 2026-03-26
tags: [symphony, harness-engineering, agentic, workflow, architecture]
---

![Sketchnote: From Harness Engineering to Symphony: The Autonomous Development Stack](/sketchnotes/articles/2026-03-26-harness-engineering-symphony.png)

*Published 2026-03-26. Sources: github.com/openai/symphony, openai.com/index/harness-engineering/, community research.*

---

On March 5, 2026, OpenAI quietly released a GitHub repository called Symphony.[^1] Within three weeks it had 13K stars.[^2] ⚠️ [unverified] The concept it embodies — **harness engineering** — is one of the most important ideas in agentic software development, and it connects directly to everything you do when you configure Codex CLI for serious use.

---

## The Problem Symphony Solves

Most teams using Codex today supervise agents directly: you open a session, describe a task, watch what happens, approve or correct, and repeat. This is valuable but it's still fundamentally human-in-the-loop at the task level.

Symphony solves a different problem: **how do you manage dozens of autonomous agents working on dozens of tasks simultaneously, without a human watching each one?**

The answer is a structured handoff model: issues go into a state called "Ready for Agent" on a Linear board.[^1] Symphony picks them up, spawns an isolated Codex agent per issue, lets it work, and only surfaces a pull request to a human when the agent has produced **proof of work**.[^1]

---

## What is Harness Engineering?

OpenAI's engineering blog (openai.com/index/harness-engineering/) introduced the term.[^3] Here's the conceptual ladder:

| Discipline | What you're designing |
|-----------|----------------------|
| Traditional engineering | Code |
| Prompt engineering | Instructions for AI |
| **Harness engineering** | The infrastructure, constraints, and feedback loops that make AI agents reliably productive |

The harness is everything that wraps the agent: AGENTS.md, test suites, build scripts, environment setup, CI pipelines, approval policies. When these are designed well, an agent can take a task and complete it autonomously with high confidence. When they're not, even a capable model will thrash, make wrong assumptions, or break things.

**Harness engineering is what you're doing every time you:**

- Write or refine your AGENTS.md
- Configure hooks (SessionStart, PostToolUse, userpromptsubmit)
- Make your test suite hermetic (no flaky external dependencies)
- Set up `codex exec` profiles for CI
- Define approval policies in your sandbox config

---

## Three Pillars of a Harness-Ready Codebase

Symphony documentation makes these requirements explicit:[^1]

### 1. Hermetic Testing

Tests that run locally, reliably, without external dependencies. An agent that runs `npm test` and gets a deterministic result can iterate on failures independently. An agent that runs tests which depend on staging databases, live APIs, or specific machine state will thrash.

**In practice:** Mock external services. Seed test databases. Make every test runnable with a single command that always works offline.

### 2. Machine-Readable Documentation

An agent should be able to discover how to build, test, and deploy your project without asking questions. This means:

```markdown
# AGENTS.md

## Build
npm run build

## Test
npm test

## Lint
npm run lint

## Deploy (staging)
npm run deploy:staging
```

If the agent can't find this, it will guess — and guessing costs tokens and introduces errors.

### 3. Modular Architecture

Codebases where side effects are minimised and boundaries are clear. An agent making a change to a payment module shouldn't risk breaking the notification service. High coupling creates correctness uncertainty; low coupling lets agents work confidently.

---

## The Symphony Workflow in Detail

```
Linear board: "Ready for Agent" state
          ↓
Symphony polls (Elixir/BEAM — can run hundreds of agents concurrently)
          ↓
Creates per-issue isolated workspace (deterministic, hermetic)
          ↓
Spawns Codex CLI agent with WORKFLOW.md as system prompt
          ↓
Agent implements task autonomously
          ↓
Proof of Work generated:
  - CI/CD status (tests green)
  - Automated code review feedback
  - Code complexity analysis
  - Walkthrough video of changes
          ↓
Human reviews PR with evidence, not supervision
```

The critical shift: humans review **verified outcomes** rather than **supervising process**. The agent doesn't just claim it's done — it demonstrates completion.

---

## WORKFLOW.md: Agent Policy as Code

Symphony introduces a new config file format that is more powerful than AGENTS.md: `WORKFLOW.md`.[^1]

```yaml
---
# YAML front matter — runtime configuration
model: gpt-5-codex
max_turns: 40
issue_tracker: linear
workspace_mode: isolated
proof_of_work: [ci, review, complexity]
---
# Markdown body — Jinja prompt template

You are implementing {{ issue.title }}.

Context:
{{ issue.description }}

Repository: {{ repo.name }}
Branch: {{ branch_name }}

Rules:
- Run the full test suite before marking done
- Do not modify {{ protected_files }}
- Write tests for any new functions
```

The YAML front matter is the runtime config (model, limits, integrations). The Markdown body is the agent's system prompt, with Jinja templating for issue-specific context.

**Why this matters:** Agent behaviour is now a **versioned repo artifact**. Changes to how your agent works are tracked in git alongside the code it produces. You can diff, review, and roll back agent policy just like application code.

---

## The Proof of Work Principle

The most important idea in Symphony is **proof of work**.[^1] An agent completing a task must demonstrate completion through verifiable evidence — not just report that it's done.

Before a PR is surfaced to a human, Symphony requires:[^1]

1. **CI green** — all tests pass in the CI pipeline
2. **Review feedback** — automated code review analysis
3. **Complexity analysis** — code change size and complexity metrics
4. **Walkthrough video** — a recording of what the agent changed and why

This addresses a fundamental trust problem with autonomous agents: how do you know the task was actually done correctly? The proof-of-work model forces accountability into the agent loop itself.

**Applying this without Symphony:** Even in a direct Codex CLI workflow, you can enforce proof-of-work thinking:

- End every subagent with: "Run the tests and report results before finishing"
- Use a `Stop` hook that checks CI status
- Require the agent to write a summary of what it changed before closing the session

---

## How This Fits Your Agentic Pod

If you're building an agentic pod with Codex, you're already doing harness engineering. Symphony shows where that road leads:

| You're doing now | Symphony's vision |
|-----------------|-------------------|
| AGENTS.md with project context | WORKFLOW.md with Jinja-templated agent policies |
| Running Codex sessions manually per task | Symphony orchestrating sessions per Linear issue |
| Reviewing agent output in the terminal | Reviewing a PR backed by CI, code review, and a walkthrough video |
| Git worktrees for isolation | Per-issue deterministic isolated workspaces |
| PostToolUse hook running tests | Proof of work enforced before PR submission |

---

## Technical Architecture: Why Elixir?

Symphony is built in Elixir (95.4% of the codebase) ⚠️ [unverified] on the Erlang/BEAM runtime.[^1] This isn't accidental:

- **Concurrency:** BEAM handles thousands of lightweight processes. Running 50 parallel Codex agents is manageable; doing this in a Python process pool would be fragile.[^4]
- **Fault tolerance:** OTP supervision trees mean one failed agent restarts automatically. A Python async loop that panics on one agent crashes everything.[^4]
- **Long-running processes:** Agents can take minutes or hours. BEAM is designed for this; HTTP-based orchestration frameworks are not.[^4]

The downside: Elixir has a smaller ecosystem and a steeper learning curve for most teams.

---

## Current Limitations

Symphony is in **engineering preview** — valuable for learning the pattern, not yet recommended for production:[^1][^5]

- Linear only (no Jira, GitHub Issues, etc.)[^1]
- Single agent per issue (no internal subagent spawning within Symphony)[^5]
- No quality gates beyond CI/tests ⚠️ [unverified]
- Requires Elixir knowledge for customisation[^1]
- No multi-repo support ⚠️ [unverified]

---

## Key Takeaway

Symphony is the most concrete vision of where agentic software development is heading: from humans supervising agents at the task level, to humans managing work at the project level while agents operate autonomously below.

The three things that make this possible — hermetic tests, machine-readable documentation, modular architecture — are the same things that make your Codex CLI setup effective today. Harness engineering is both the practice and the path.

---

*See also:*

- *[notes/symphony.md](/codex-resources/notes/symphony/) — reference note on Symphony*
- *[articles/2026-03-26-codex-cli-cicd-non-interactive.md](/codex-resources/articles/2026-03-26-codex-cli-cicd-non-interactive/) — Codex in CI/CD*
- *[articles/2026-03-26-codex-cli-hooks-deep-dive.md](/codex-resources/articles/2026-03-26-codex-cli-hooks-deep-dive/) — hooks for proof-of-work patterns*

---

## Citations

[^1]: [OpenAI Symphony — GitHub Repository](https://github.com/openai/symphony)
[^2]: Star count not independently verified at time of writing; check the repository directly for the current figure.
[^3]: [Harness engineering: leveraging Codex in an agent-first world — OpenAI Engineering Blog (Ryan Lopopolo)](https://openai.com/index/harness-engineering/)
[^4]: [OpenAI Releases Symphony: An Open Source Agentic Framework — MarkTechPost](https://www.marktechpost.com/2026/03/05/openai-releases-symphony-an-open-source-agentic-framework-for-orchestrating-autonomous-ai-agents-through-structured-scalable-implementation-runs/)
[^5]: [OpenAI Symphony SPEC.md — Limitations and single-agent-per-issue design](https://github.com/openai/symphony/blob/main/SPEC.md)
