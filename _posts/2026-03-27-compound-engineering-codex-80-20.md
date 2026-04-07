---
title: "Compound Engineering with Codex: The 80/20 Plan-Review Model"
subtitle: "Invert the traditional development ratio — spend 80% on planning and review, 10% on execution, and 10% on compounding knowledge"
date: 2026-03-27
tags: [compound-engineering, multi-agent, review, planning, codex-cli, philosophy]
---
![Sketchnote: Compound Engineering with Codex: The 80/20 Plan-Review Model](/sketchnotes/articles/2026-03-27-compound-engineering-codex-80-20.png)

*Based on: every.to/guides/compound-engineering · github.com/EveryInc/compound-engineering-plugin · notes/compound-engineering.md*

---

## The Core Problem

Most developers using Codex CLI still work in the old ratio: write a prompt, watch the agent execute, review the output, repeat. The distribution looks like:

- 10% planning
- 80% execution
- 10% review

This mirrors pre-AI development rhythms. The developer has stepped back from writing code, but the cognitive distribution hasn't changed. They're still reactive — watching execution rather than steering cycles.

Compound engineering inverts this deliberately.

---

## The 80/20 Model

The EveryInc team, who builds Codex tooling professionally, advocate a ratio that looks more like:

| Phase | Traditional | Compound Engineering |
|-------|------------|---------------------|
| Planning | 10% | **40%** |
| Execution | 80% | **10%** |
| Review | 10% | **40%** |
| Knowledge codification | ~0% | **10%** |

The agents handle the 10% execution. Humans contribute where they're irreplaceable: deciding *what* to build (planning) and validating *whether it's right* (review).

The tenth percent — knowledge codification — is the "compound" that makes each subsequent cycle faster. Without it, you're on a treadmill. With it, you're building a flywheel.

---

## The Compound Engineering Loop

The compound-engineering-plugin (`@every-env/compound-plugin`) ships six named commands that map directly onto this cycle:

```
/ce:brainstorm → /ce:plan → /ce:work → /ce:review → /ce:compound → /ce:ideate
```

### `/ce:brainstorm`
Dialogue-driven requirements refinement. The agent asks clarifying questions, surfaces edge cases, and converts rough ideas into structured specifications. The goal: never start execution with an ambiguous brief.

### `/ce:plan`
Requirements become a detailed technical specification — architecture decisions, interface contracts, test strategies, potential risks — all *before* a line of code is written. The plan is a versioned artefact, committed to the repo.

### `/ce:work`
Agents implement while developers monitor, not code. In **v2.54.1 (March 2026)**, `ce:work` now supports **Codex delegation mode**: the compound plugin handles orchestration; Codex CLI handles execution. Clear separation of concerns.

### `/ce:review`
This is where the 40% review investment concentrates. The plugin dispatches **14 specialised reviewers in parallel**:

- Security auditor
- Performance analyst
- Architecture reviewer
- Accessibility checker
- Test coverage analyser
- API design reviewer
- Documentation reviewer
- (and 7 more)

Each runs simultaneously. A complex PR that would take a senior engineer an hour to review — in full, with all dimensions — takes minutes. The output is structured, actionable, and comprehensive.

This is the "26 agents in parallel" pattern you'll see cited in the Codex community.

### `/ce:compound`
The step most teams skip — and the one that makes the philosophy work. After a successful cycle:

- Document what worked and what didn't
- Codify decisions into AGENTS.md rules
- Add patterns to skill files
- Update test templates
- Commit institutional knowledge into the system

Every hour spent here reduces future cycles. This is the mechanism that turns a tool into a capability.

### `/ce:ideate`
Not prompted by a specific task — proactively surfaces where compound investments would pay off in the existing codebase. Think of it as the agent reading your codebase and suggesting where to codify taste before you need it.

---

## Installing with Codex CLI

```bash
# Install
bunx @every-env/compound-plugin install compound-engineering --to codex

# Update to latest
bunx @every-env/compound-plugin sync --target codex
```

The plugin ships as a standard Codex skill bundle, so all commands appear in the `$` skill namespace.

**Hook integration (in progress as of March 2026):** PR #354 adds hooks for all five Codex lifecycle events — `SessionStart`, `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, and `Stop` — enabling the compound loop to trigger automatically rather than requiring manual `/ce:*` invocations.

---

## The Four Principles Behind the Model

**1. Taste belongs in systems.**
Formatting preferences, naming conventions, architectural decisions — all of these should be codified in AGENTS.md, CI rules, and skill files rather than applied via manual review. Every rule you write is a review cycle you'll never have to have again.

**2. Teach the system.**
Time invested giving agents context returns exponentially. An agent that knows your codebase, your conventions, and your preferences doesn't require hand-holding. The `/ce:compound` step is how you teach.

**3. Build safety nets.**
Verification infrastructure replaces gatekeeping. Tests, CI, automated review — the safety net that lets both agents and developers move fast without fear. The review step isn't optional; it's structural.

**4. Make environments agent-native.**
Agents should access everything a senior engineer can access: APIs, databases, issue trackers, CI outputs, documentation. MCP servers are the mechanism. If your agent is working blind, it's working at a disadvantage.

---

## Connection to Codex's Own Architecture

Compound engineering maps naturally onto Codex CLI's native capabilities:

| Compound Principle | Codex Mechanism |
|-------------------|-----------------|
| Taste in systems | AGENTS.md rules + skills |
| Teach the system | Context files, project-doc fallbacks |
| Safety nets | Hooks (PostToolUse), CI integration, `codex exec` |
| Agent-native environment | MCP servers, web_search config |
| Parallel review | Subagents + spawn_agents_on_csv |

The `/ce:review` parallel reviewer pattern — 14 agents simultaneously analysing a PR — is directly implementable in Codex using subagents:

```toml
# In a subagent config
[subagent.security]
prompt = "Review the diff at $DIFF_PATH for security vulnerabilities. Focus on: injection, auth bypass, secrets exposure."

[subagent.performance]  
prompt = "Review the diff at $DIFF_PATH for performance issues. Focus on: N+1 queries, unnecessary allocations, blocking I/O."

# ... (repeat for each reviewer dimension)
```

The `spawn_agents_on_csv` feature (updated in March 2026 with progress tracking and ETA estimates) makes this pattern tractable at scale.

---

## Relationship to Symphony's Proof of Work

Both compound engineering and Symphony (Harness Engineering's autonomous delivery layer) converge on the same insight: agents need structured verification before work is accepted.

| Symphony | Compound Engineering |
|---------|---------------------|
| CI green | Safety nets (principle 3) |
| PR review feedback | `/ce:review` parallel reviewers |
| Walkthrough video | `feature-video` skill |
| Proof of work gate | Review phase gate |

The difference: Symphony enforces this at the platform level; compound engineering applies it as a team discipline. Both are heading to the same place.

---

## Practical Starting Points

If you're adopting this incrementally:

1. **Start with `/ce:review`** — the parallel review step delivers immediate, measurable value with no change to your existing workflow. Install the plugin, run `/ce:review` on your next non-trivial PR.

2. **Add `/ce:plan` next** — the planning step reduces back-and-forth mid-execution and makes the review step more meaningful (you're checking against the plan).

3. **Build the `/ce:compound` habit** — after your first successful agent cycle, spend 15 minutes writing down what worked. Commit it to AGENTS.md. Repeat until it's automatic.

4. **Then wire it to hooks** — once PR #354 ships with full hook support, set `PostToolUse` hooks to trigger automatic review on significant file writes.

---

## See Also

- [notes/compound-engineering.md](/codex-resources/notes/compound-engineering/) — extended research notes
- [articles/2026-03-27-multi-agent-orchestration-patterns.md](/codex-resources/articles/2026-03-27-multi-agent-orchestration-patterns/) — parallel subagent patterns
- [articles/2026-03-27-proof-of-work-principle-agents.md](/codex-resources/articles/2026-03-27-proof-of-work-principle-agents/) — Symphony's related proof-of-work concept
- [resources/ecosystem-tools.md](/codex-resources/resources/ecosystem-tools/) — compound-engineering-plugin technical details
