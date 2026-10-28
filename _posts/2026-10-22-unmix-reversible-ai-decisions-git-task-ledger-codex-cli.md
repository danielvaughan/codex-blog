---
title: "Unmix: Reversible AI Decisions and the Git Task Ledger Pattern"
date: 2026-10-22T08:00:00+00:00
last_modified_at: 2026-10-28T10:24:08Z
parent: "Articles"
nav_order: 1195
tags:
  - codex-cli
  - community
  - unmix
  - mcp
  - git
  - reversibility
  - multi-agent
  - open-source
copy_edited: true
---

# Unmix: Reversible AI Decisions and the Git Task Ledger Pattern

![Sketchnote diagram for: Unmix: Reversible AI Decisions and the Git Task Ledger Pattern](/sketchnotes/articles/2026-10-22-unmix-reversible-ai-decisions-git-task-ledger-codex-cli.png)

**Community tool:** Unmix (GitHub issue #53955, OpenAI Developer Community)
**Pattern:** Git task ledger + MCP for reversible agent decisions
**Status:** Active development, October 2026
**Category:** Multi-agent workflow infrastructure

---

One of the persistent anxieties in agentic coding is commitment anxiety: the feeling that an agent is making decisions faster than you can evaluate them, and that by the time you notice a bad decision, the workspace is in a state that is difficult to undo.

This is not a new problem in software. Version control solves it for human-authored code. But AI-authored changes arrive in a different pattern: multiple files modified simultaneously, often with interdependencies, in response to a high-level instruction whose interpretation you cannot fully control. A `git revert` undoes a commit. It does not undo the reasoning that led to the commit, or the subsequent decisions that built on that reasoning.

Unmix, a community-developed Codex tool, is built around a different approach: make agent decisions reversible at the task level, not just the file level. The mechanism is a Git task ledger — a structured record, stored in the repository itself, that captures each agent decision as a discrete, addressable entry. Combined with MCP, the ledger becomes a live interface that allows Codex to query, annotate, and reverse decisions as part of its normal workflow.

---

## The Commit Is Not the Decision

The insight that motivates Unmix begins with a distinction that version control does not make: the difference between a code change and the decision to make that change.

When an agent modifies a file, git records the modification. What it does not record is:
- Why the agent made the change
- What alternatives it considered and rejected
- What other decisions depend on this change
- Whether the change was confident or uncertain

These four things matter enormously when you want to reverse a decision. A simple `git revert` undoes the modification but leaves the reasoning intact, in a form that may cause the agent to recreate the same decision if re-prompted. It also does not tell you which subsequent changes depended on the reverted decision — changes that may now be logically inconsistent with the reverted state.

The Git task ledger is Unmix's answer to this gap.

---

## The Git Task Ledger

Unmix adds a structured YAML file to the repository — by convention, `.codex/task-ledger.yaml` — that records every significant agent decision as an entry:

```yaml
tasks:
  - id: task-0041
    timestamp: "2026-10-20T14:23:11Z"
    instruction: "Add OAuth2 token refresh to the auth service"
    agent_session: "session-a7f2"
    files_modified:
      - src/auth/token.ts
      - src/auth/refresh.ts
      - tests/auth/token.test.ts
    decision_log:
      - "Chose sliding-window TTL over fixed expiry based on existing session management pattern"
      - "Deferred refresh rate limiting — flagged as follow-up (see task-0042)"
    confidence: high
    reversible: true
    depends_on: []
    depended_on_by:
      - task-0042
      - task-0045
    status: committed

  - id: task-0042
    timestamp: "2026-10-20T15:01:47Z"
    instruction: "Add refresh rate limiting flagged in task-0041"
    agent_session: "session-a7f2"
    files_modified:
      - src/auth/rate-limit.ts
      - src/middleware/auth.ts
    decision_log:
      - "Used token-bucket algorithm to match existing rate limiter in api/ directory"
    confidence: medium
    reversible: true
    depends_on:
      - task-0041
    depended_on_by: []
    status: committed
```

Each entry captures the instruction, the files modified, the agent's decision rationale, a confidence level, and — critically — the dependency graph: which tasks this task depends on, and which tasks depend on it.

---

## Reversing a Decision

The power of the dependency graph becomes apparent when you want to undo a decision. If you want to reverse `task-0041` (the OAuth2 token refresh), Unmix does not just `git revert` the associated commits. It shows you the dependency chain:

```text
$ unmix reverse task-0041

Task task-0041: "Add OAuth2 token refresh to the auth service"
  Modified: src/auth/token.ts, src/auth/refresh.ts, tests/auth/token.test.ts

⚠️  This task has dependents:
  → task-0042: "Add refresh rate limiting flagged in task-0041"
     Modified: src/auth/rate-limit.ts, src/middleware/auth.ts

Reversing task-0041 will leave task-0042 in an inconsistent state.

Options:
  [1] Reverse both task-0041 and task-0042 (recommended)
  [2] Reverse task-0041 only and flag task-0042 for review
  [3] Cancel

Choice:
```

This is meaningfully different from `git revert`. You are not just undoing a file change — you are resolving a decision graph. The dependent task (task-0042) was explicitly built on task-0041's work. Reversing task-0041 without addressing task-0042 would leave rate-limiting code that references a token refresh mechanism that no longer exists.

Unmix makes this visible before you act, not after.

---

## The MCP Interface

Unmix exposes the task ledger as an MCP server. This turns the ledger from a static record into a live tool that Codex can query during its own execution.

This enables two new patterns:

### 1. Self-aware agent decisions

Before making a significant change, an agent can query the ledger:

```text
[Codex, via MCP]
tool: unmix_query_ledger
query: "Are there any existing decisions about the auth service token handling?"

response:
  matching_tasks:
    - task-0041: "Add OAuth2 token refresh" — confidence: high, status: committed
    - task-0042: "Add refresh rate limiting" — confidence: medium, status: committed

  note: "Any modification to src/auth/token.ts should account for the existing refresh
         mechanism documented in task-0041."
```

The agent is informed of prior decisions before it makes a new one. This is the pattern of explicit context that [Goal Instruction Writer](../2026-10-21-codex-community-agent-handoff-tools-ctx-kit-estafeta-goal-instruction-writer/) attempts to solve via AGENTS.md — but Unmix solves it dynamically, via a queryable live record, rather than statically via a configuration file.

### 2. Agent-initiated reversal

An agent can request a reversal:

```text
[Codex, via MCP]
tool: unmix_reverse
task_id: task-0041
reason: "Revised approach — using session-level caching instead of per-request refresh"

response:
  dependents:
    - task-0042 (will be flagged for review)
  files_to_revert:
    - src/auth/token.ts (reverting to commit a3f21b)
    - src/auth/refresh.ts (reverting to commit a3f21b)
    - tests/auth/token.test.ts (reverting to commit a3f21b)
  confirmation_required: true
```

Human approval is required before any reversal executes. The agent can propose; the human decides. This is the correct model for agent-initiated changes to the codebase history — the agent should never be able to reverse past decisions autonomously.

---

## What Unmix Is Not

Unmix does not replace git. It works alongside git, using git as its storage mechanism (the task ledger is a file in the repository) and its reversal mechanism (reverts are implemented as git reverts). Git provides the ground truth of what changed. Unmix provides the layer above: what the agent decided, why, and how decisions relate to each other.

Unmix does not prevent bad agent decisions. An agent can still make confident, incorrect, consequential changes. What Unmix changes is the cost of recovering from them: from a manual archaeology expedition through git history to a structured query against a decision graph.

Unmix does not work across repositories. The task ledger is scoped to a single repository. Multi-repo agent workflows are a harder problem.

---

## Why This Pattern Matters

The Git task ledger pattern addresses something that has been absent from every agentic coding tool: a native concept of decision provenance.

Human developers using git can reconstruct why a change was made by reading the commit message and, if they were lucky, the PR description. AI agents do not write good commit messages — they write whatever they were configured to write. And they do not write PR descriptions that capture the reasoning behind design choices.

Unmix is a prototype of a pattern that should probably be native to agentic coding tools: every significant agent action should be recorded with its rationale, its alternatives considered, its confidence level, and its dependencies. Not in a chat history that scrolls off the screen, but in a structured record in the repository that can be queried and acted on.

The MCP integration is the right architecture for this. The task ledger becomes a tool capability that any agent can use, not a bespoke interface for a specific workflow. Codex can query it. A future agent can query it. A developer reading the repository six months later can query it.

---

## Getting Started

Unmix is available as a community tool:

```bash
# Install Unmix
npm install -g unmix-codex

# Initialise the task ledger in your repository
unmix init

# Add the MCP server to your Codex configuration
# Add to ~/.codex/config.json:
# {
#   "mcpServers": {
#     "unmix": {
#       "command": "unmix",
#       "args": ["mcp"]
#     }
#   }
# }
```

The `.codex/task-ledger.yaml` file should be committed to source control. It is the record — treat it as such.

---

## The Upstream Question

Unmix's Git task ledger pattern is architecturally sound and practically useful. The question is whether it will remain a community tool or be upstreamed into Codex CLI itself.

The case for upstreaming is strong: decision provenance is a universal concern in agentic coding, not a niche requirement. Every team running Codex on production code eventually asks "why did the agent do that?" Unmix's answer is the right kind of answer — structured, queryable, actionable.

The case against immediate upstreaming is equally clear: Unmix is a prototype. Its schema is unstable. Its MCP integration is early. The right moment to upstream is after the pattern has been validated by community use, not before.

Watch this space. The Git task ledger may be the most undervalued pattern in the Codex ecosystem right now.

---


[^1]: Unmix, GitHub issue #53955, OpenAI Developer Community, October 2026. Community tool for reversible AI decisions via MCP + Git task ledger.
[^2]: Codex community weekly report, 22 October 2026. Unmix flagged as "architecturally novel — article candidate."
[^3]: Goal Instruction Writer, previously covered in this series, 21 October 2026.
[^4]: MCP as tool interface for Codex: v0.164.0 enterprise MCP auth overhaul, covered 21 October 2026.
