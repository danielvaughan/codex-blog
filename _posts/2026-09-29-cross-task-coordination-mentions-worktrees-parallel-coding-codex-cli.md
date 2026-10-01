---
title: "Cross-Task Coordination: Combining @Mentions and Native Worktrees for Parallel Coding"
date: 2026-09-29T08:00:00+00:00
last_modified_at: 2026-10-01T12:28:47+01:00
tags: ["codex-cli", "multi-agent", "worktrees", "at-mentions", "parallel-coding", "v0.150.0", "v0.154.0", "v0.156.0", "cross-task", "orchestration"]
---

# Cross-Task Coordination: Combining @Mentions and Native Worktrees for Parallel Coding


---

Two features shipped within six weeks of each other in mid-2026 and, together, change how parallel Codex CLI sessions coordinate. The `@task-name` inter-session messaging primitive arrived in v0.150.0 STABLE (26 August 2026).[^1] The `WorktreeManager` API for native git worktree isolation landed in v0.154.0 (September 2026),[^5] with worktree support promoted to on-by-default in v0.156.0 (22 September 2026).[^2] Neither feature alone is remarkable — cross-session messaging existed in nascent form before v0.150.0, and manual `git worktree add` scripting predates Codex CLI entirely. The combination, however, creates something new: parallel sessions that work in genuine isolation and can still communicate structured results to each other without file-merging risk.

This article covers the mechanics of both features and three workflow patterns that compose them.

## What Each Feature Does

### @task-name mentions

When you reference another running or paused Codex task by name using `@task-name` syntax, the session can read that task's output, status, and history. You can also send a message to another task, inject a question mid-session, or trigger a coordination step that depends on another session's result.

Before v0.150.0, cross-session coordination required shared files, manual polling, or the lower-level `codex queue` message bus — approaches that either introduced file-merge risk or required scripting. The `@mention` primitive gives agents a direct, named channel to query each other without touching the filesystem.

### Native worktree isolation

Before `WorktreeManager::create` landed in v0.154.0, running two Codex sessions in parallel on the same repository required manually scripting `git worktree add`, suppressing inherited environment variables (`GIT_DIR`, `GIT_WORK_TREE`, `GIT_INDEX_FILE`), disabling content filters, and handling rollback on failure. This was well-documented in the community but error-prone and not beginner-accessible.[^3]

`WorktreeManager` encapsulates the full isolation contract behind a single managed call. Each parallel session gets a detached, Desktop-compatible worktree from HEAD or an explicit base ref. Hook inheritance, filesystem monitors, and environment variables are suppressed automatically. If setup fails, the manager rolls back. With v0.156.0 enabling worktrees by default, the agents command centre can create a worktree session directly from the task list — no manual scripting required.

## The Before/After Ergonomics Gap

The table below contrasts the manual approach with the current managed approach:

| Concern | Manual (pre-v0.154.0) | Managed (v0.154.0+, default v0.156.0) |
|---|---|---|
| Create parallel session | `git worktree add .worktrees/task-b HEAD` | Agents command centre or `codex agents` TUI |
| Suppress env vars | Export 4+ git env vars explicitly | Handled automatically by `WorktreeManager` |
| Hook suppression | Must unset or disable manually | Automatic |
| Rollback on failure | Manual `git worktree remove` | Automatic |
| Cross-session query | Shared temp file or `codex queue` bus | `@task-name` mention |
| Merge responsibility | Implicit — easy to forget | Explicit human merge step by design |


The practical effect: a two-worktree parallel session that previously required a ten-line shell setup script now takes four keystrokes in the agents TUI. The `@mention` communication channel replaces ad-hoc file passing with named, queryable session references.

## Three Workflow Patterns

### Pattern 1: Investigation/Implementation Split

The most common cross-task pattern separates diagnosis from repair. Task A (`@investigate`) runs in read-only mode, analyses the failing test surface, and produces a structured diagnosis. Task B (`@implement`) waits for a signal from Task A before writing any code, then queries `@investigate` to confirm which files are in scope and what the root cause is before proceeding.

```text
# In Task B (implement), after investigation is complete:
# "Before modifying anything, query @investigate for the root cause summary."
# The agent issues: @investigate — what files are affected and what is the root cause?
```

The critical property here is sequencing: Task B is not blocked at the system level, but the `@mention` query before any file write creates a soft dependency that prevents premature file merging. When Task A completes, Task B can reference its conclusion and proceed without the human needing to copy output between terminals.

This pattern is useful when the diagnosis phase involves many read operations that would pollute the implementation session's context window. Keeping them separate preserves reasoning quality in the implementation session.

### Pattern 2: A/B Architectural Comparison

Two worktrees implement the same feature using different architectural approaches. A third session (`@compare`) queries both via `@mentions` when they complete, summarises the tradeoffs, and surfaces the comparison for a human decision before either branch is merged.

```text
# Session structure:
# @approach-a  — implements Option A in worktree-a
# @approach-b  — implements Option B in worktree-b
# @compare     — queries @approach-a and @approach-b, then summarises

# In @compare:
# "@approach-a — summarise your implementation and identify any constraints.
#  @approach-b — do the same. Then compare the two."
```

The comparison session runs after both implementations signal completion. Because each approach is in its own worktree, there is no file-level conflict. The human reviews the `@compare` summary, selects one branch, and merges it. The other worktree is discarded.

This pattern works well for architecture decisions where running both options in parallel is faster than sequential evaluation. The `@mention` channel means the comparison agent does not need to re-read both codebases from scratch — it pulls structured summaries directly from the implementation sessions.

### Pattern 3: Front-End/Back-End Parallel Sprint

Two sessions develop front-end and back-end components in parallel. Midway through, a contract definition — an API interface or type schema — must be agreed before either session can complete its work. The `@mention` channel synchronises this without stopping either session.

```text
# @frontend and @backend run in parallel.
# At the agreed contract checkpoint:
# @frontend queries: "@backend — confirm the /api/events response schema."
# @backend replies with the current schema draft.
# @frontend continues with the confirmed schema.
```

Without `@mentions`, this synchronisation point requires the human to copy the schema definition between terminals or commit it to a shared branch. The `@mention` channel handles the handoff inside the agent sessions themselves — the human's role is to watch for the contract checkpoint and confirm it looks right before each session proceeds past it.

The key design decision for this pattern is identifying the contract checkpoint in advance. It belongs in the AGENTS.md under a `## Coordination` section, listing the task name, the checkpoint name, and the expected information exchange.[^4]

## What the Explicit Merge Step Means

A completed worktree session does not guarantee clean integration with any other session. Two agents completing their respective implementations does not produce a merged codebase — it produces two isolated branches. The human merge step is structural, not optional.

This is by design. Codex CLI's worktree model treats git merge as a deliberate human action, not an automatic consequence of agent completion. The AGENTS.md for any multi-worktree workflow should include an explicit merge discipline section:

```markdown
## Merge Discipline

- No session merges its own worktree to main.
- Human reviews both branches before merging.
- @compare session summarises tradeoffs; human selects the branch.
- Unused worktrees are cleaned up via the agents command centre after merge.
```

This makes the integration responsibility visible rather than implicit. Teams that skip documenting the merge step tend to accumulate stale worktrees and discover integration conflicts at a point where both implementations have diverged significantly.

## AGENTS.md Template for a Two-Session Parallel Sprint

```markdown
## Parallel Session Configuration (v0.156.0+)

### Active tasks
| Task name    | Worktree        | Role         | Status   |
|--------------|-----------------|--------------|----------|
| @investigate | worktree-invest | Read-only    | Running  |
| @implement   | worktree-impl   | Write        | Pending  |

### Coordination
- @implement must query @investigate before any file write.
- Contract checkpoint: @implement confirms affected files with @investigate.
- Merge: Human reviews both sessions, then merges @implement branch.

### Merge discipline
- Neither task merges to main.
- @implement uses: git diff main to confirm scope before final commit.
```

## Summary

The `@task-name` mention primitive and the managed worktree API are individually useful. Combined, they enable a coordination model that was previously only achievable with substantial setup scripting. Three patterns cover most parallel session needs: investigation/implementation split for diagnosis-before-repair workflows, A/B comparison for architecture decisions, and front-end/back-end sprints with mid-session contract synchronisation.

The invariant across all three patterns is the same: agent completions do not produce an integrated result automatically. The explicit human merge step is not a limitation of the feature — it is the point at which the human's judgment replaces the agent's.

---

**Companion articles:**
- [Agentic Workspace Taxonomy: From Terminal Tabs to Purpose-Built Orchestration](2026-09-22-agentic-workspace-taxonomy-terminal-tabs-orchestration-codex-cli.md)
- [AGENTS.md as Multi-Agent Orchestration Manifest](2026-09-21-agents-md-multi-agent-orchestration-manifest-dependency-annotations-exclusion-zones-codex-cli.md)
- [Decision-Impact Scoring for Codex Queue](2026-09-21-decision-impact-scoring-codex-queue-task-interdependency-metadata-codex-cli.md)

---

[^1]: OpenAI, "Codex CLI v0.150.0 Release Notes," GitHub, 26 August 2026. https://github.com/openai/codex/releases/tag/v0.150.0

[^2]: OpenAI, "Codex CLI v0.156.0 Release Notes," GitHub, 22 September 2026. https://github.com/openai/codex/releases/tag/v0.156.0

[^3]: lustoykov, "Running Multiple Codex Sessions in Parallel with git worktree," Codex CLI Community, March 2026. https://community.openai.com/t/running-multiple-codex-sessions-with-git-worktree/

[^4]: Vaughan, D., "AGENTS.md as Multi-Agent Orchestration Manifest," danielvaughan.com/codex-resources, 21 September 2026. https://danielvaughan.com/codex-resources/articles/2026-09-21-agents-md-multi-agent-orchestration-manifest-dependency-annotations-exclusion-zones-codex-cli/

[^5]: OpenAI, "Codex CLI v0.154.0 Release Notes," GitHub, September 2026. https://github.com/openai/codex/releases/tag/v0.154.0
