---
title: "Worktree-Based Parallel Development with Codex CLI"
layout: single
date: 2026-03-26
---
![Sketchnote: Worktree-Based Parallel Development with Codex CLI](/sketchnotes/articles/2026-03-26-codex-cli-worktree-parallel-development.png)

# Worktree-Based Parallel Development with Codex CLI

*Published 2026-03-26 | Sources: developers.openai.com/codex/app/worktrees, nimbalyst.com, github.com/openai/codex*

---

## Why Worktrees?

The core limitation of sequential AI agent workflows: **one agent, one thread, one context**. Every task blocks the next. When Codex is investigating a bug, you can't also be writing a new feature in the same directory — the agent would interfere with itself.

Git worktrees solve this. A worktree is a second (or third, or fourth) working directory linked to the same git repository.[^1] Each worktree has its own branch, its own index, and its own file state — but all share the same commit history.[^1] This makes them ideal for parallel agent execution:

- No file conflicts between agents running simultaneously
- Clean branch isolation — each agent works on its own feature branch
- Easy merge review — each worktree produces a discrete diff

> *"Git worktrees are the single most important technique for parallel AI agent development."* — Nimbalyst[^2]

---

## Worktrees in the Codex App vs CLI

### Codex App (built-in support)

The Codex desktop app has **first-class worktree support** built in.[^3] When starting a new thread:

1. Select **"Worktree"** in the new thread view[^3]
2. Choose the starting branch (main, feature branch, or current branch with local changes)[^3]
3. Submit your prompt — Codex creates the worktree in **detached HEAD state** (prevents branch pollution while supporting multiple parallel worktrees)[^3]
4. Codex applies any uncommitted local changes to the worktree automatically[^3]

**Two paths forward:**

- **Path A — Exclusive Worktree Development**: Click "Create branch here" to convert the detached worktree into a named branch. Commit, push, and open PRs directly from the app.[^3]
- **Path B — Handoff to Local**: Click "Hand off" in the thread header to bring work back to your main checkout. Best when you need your dev server, IDE, or specific local env.[^3]

**Automations with worktrees** (App 26.312+)[^4]: When configuring automations, choose "Worktree" execution per automation. Each scheduled run gets a clean isolated worktree — no cross-contamination between automation runs.[^4]

### Codex CLI (manual worktree management)

The CLI does not yet have a `--worktree` flag in stable release (GitHub issues [#13120](https://github.com/openai/codex/issues/13120)[^5] and [#12862](https://github.com/openai/codex/issues/12862)[^6] track this). Until it ships, manage worktrees manually:

```bash
# Create a new worktree with a new branch for a parallel agent task
git worktree add ../myapp-feature -b feat/add-payments

# Start Codex in that worktree
cd ../myapp-feature && codex "Implement Stripe checkout integration per the spec in docs/payments.md"

# In another terminal — start a second agent on a different task
git worktree add ../myapp-bugfix -b fix/auth-token-expiry
cd ../myapp-bugfix && codex "Investigate and fix the token expiry bug in src/auth/session.ts"

# Check all active worktrees
git worktree list

# Clean up after merging
git worktree remove ../myapp-feature
```

**Rule of thumb:** Limit to 3-4 parallel CLI sessions. Review and merge become the bottleneck beyond that.

---

## Subagents and Worktree Isolation

When Codex spawns subagents, each agent works in its own isolated environment — often a git worktree created automatically by the subagent runtime. This is key to how parallel subagent workflows avoid file conflicts.

As of **v0.115.0** (March 16, 2026)[^7], spawned subagents now reliably inherit:

- Sandbox and network rules from the parent session[^7]
- Project-profile layering (`.codex/config.toml` profiles)[^7]
- Persisted host approvals[^7]
- Symlinked writable roots[^7]

This means subagent-driven parallel execution is now stable for automated/CI use cases.

### Agentic Pod Pattern with Worktrees

A typical multi-agent pattern for Daniel's agentic pod:

```
Orchestrator (main thread, main checkout)
├── Agent 1 — worktree: feat/api-endpoints    (new feature)
├── Agent 2 — worktree: fix/perf-regression   (bug fix)
└── Agent 3 — worktree: chore/update-deps     (dependency update)
```

Each agent runs independently. The orchestrator reviews and merges when complete.

---

## Key Constraints and Gotchas

| Constraint | Detail |
|---|---|
| **One branch, one worktree** | Git prevents the same branch from being checked out in more than one worktree — hard git rule, not Codex limitation[^1] |
| **No .gitignore files** | Files in `.gitignore` do NOT transfer during app Handoff — not git-tracked[^3] |
| **Separate node_modules** | Each worktree needs its own `npm install` / `uv sync` |
| **Detached HEAD by default** | Codex app uses detached HEAD to allow parallel worktrees on the same starting branch — convert to a named branch when you want to keep the work[^3] |
| **15 worktree limit** | Codex app defaults to retaining 15 recent managed worktrees; permanent worktrees are never auto-deleted[^3] |

---

## Nimbalyst: Visual Worktree Management

**Nimbalyst** ([nimbalyst.com](https://nimbalyst.com))[^8] is the primary GUI tool for visual worktree-based parallel development with Codex and Claude Code. It replaces the deprecated Crystal (Feb 2026)[^9].

**What it automates:**

- Creates a git worktree per session automatically (no manual `git worktree add`)[^8]
- Kanban board for managing parallel sessions[^8]
- Unified diff review pane for all agent file changes[^8]
- MCP integration (Linear, GitHub, Slack, databases)[^8]
- Mobile companion for monitoring + voice input[^8] ⚠️ [unverified: voice input specifically not confirmed in official docs]
- Race condition hardening: mutex locks, session validation, database transactions ⚠️ [unverified]

**Why it matters for the agentic pod:** Nimbalyst is essentially a Codex session manager with git worktrees as the primary isolation primitive. It's the tool that makes the "parallel agents on parallel branches" workflow tractable at scale.

**Reference:** [nimbalyst.com/blog/git-worktrees-for-ai-coding-agents-complete-guide](https://nimbalyst.com/blog/git-worktrees-for-ai-coding-agents-complete-guide/)[^2]

---

## AGENTS.md for Worktree Workflows

Add this to your repo's `AGENTS.md` to help Codex understand the worktree context:

```markdown
## Parallel Development Notes
This repo uses git worktrees for parallel agent development.
- Each agent task runs in its own worktree with its own branch
- Never push directly to main — always open a PR
- Run `npm install` (or `uv sync`) at the start of any new worktree session
- Branch naming: feat/<slug>, fix/<slug>, chore/<slug>
- After completing work, run the full test suite before handing off for review
```

---

## CI/CD Integration

Worktrees pair naturally with `codex exec` for CI:

```bash
# In CI: create isolated worktree, run agent, clean up
git worktree add /tmp/codex-ci-run -b ci/auto-fix-$(date +%s)
cd /tmp/codex-ci-run
codex exec --profile ci "Fix all failing tests and summarize changes"
# Review output, open PR if successful
git worktree remove /tmp/codex-ci-run
```

For GitHub Actions, see [`openai/codex-action`](https://github.com/openai/codex-action)[^10] — the official action already handles sandbox isolation, so explicit worktree management is optional in that context.

---

## When to Use Worktrees vs Single-Thread

| Scenario | Approach |
|---|---|
| Simple bug fix in one file | Single thread, no worktree needed |
| New feature spanning 5+ files | Worktree — isolates changes |
| Parallel feature development by multiple agents | Worktrees — one per agent |
| Overnight automation runs | Worktrees via automations (App 26.312+)[^4] |
| Exploring two implementation approaches | Two worktrees — compare diffs, merge the winner |
| CI/CD automated task | `codex exec` — sandbox handles isolation |

---

*This article corresponds to backlog item #11.*

## Citations

[^1]: Git worktree documentation — commands, constraints, and one-branch-per-worktree rule: https://git-scm.com/docs/git-worktree
[^2]: Nimbalyst blog — "Git Worktrees for AI Coding Agents: A Complete Guide" (source of the quoted statement): https://nimbalyst.com/blog/git-worktrees-for-ai-coding-agents-complete-guide/
[^3]: Official Codex app worktrees documentation — detached HEAD, handoff, .gitignore limitation, 15 worktree limit: https://developers.openai.com/codex/app/worktrees
[^4]: Codex developer changelog — App version 26.312 (March 12, 2026) introducing worktree-per-automation option: https://developers.openai.com/codex/changelog
[^5]: GitHub issue #13120 — "Git worktrees with codex" (closed as duplicate of #12862): https://github.com/openai/codex/issues/13120
[^6]: GitHub issue #12862 — "CLI: add --worktree and --tmux flags for one-command isolated sessions" (open): https://github.com/openai/codex/issues/12862
[^7]: GitHub release rust-v0.115.0 (March 16, 2026) — subagent sandbox/network inheritance, project-profile layering, persisted host approvals, symlinked writable roots: https://github.com/openai/codex/releases/tag/rust-v0.115.0
[^8]: Nimbalyst homepage — visual workspace for Claude Code and Codex with kanban, diff review, MCP integration, mobile app: https://nimbalyst.com/
[^9]: Crystal (stravu/crystal) deprecated and replaced by Nimbalyst as of February 2026: https://github.com/stravu/crystal
[^10]: openai/codex-action — official GitHub Actions action for running Codex CLI in CI with sandbox isolation: https://github.com/openai/codex-action
