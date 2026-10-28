---
title: "Worktrees Without the Flag: Migrating to v0.163.0's Default-On Managed Worktrees"
parent: "Articles"
nav_order: 1200
date: 2026-10-21T13:00:00+00:00
last_modified_at: 2026-10-28T13:04:38Z
tags:
  - codex-cli
  - v0.163.0
  - git-worktrees
  - managed-worktrees
  - config-migration
  - configuration
  - feature-flags
copy_edited: true
---

# Worktrees Without the Flag: Migrating to v0.163.0's Default-On Managed Worktrees

![Sketchnote diagram for: Worktrees Without the Flag: Migrating to v0.163.0's Default-On Managed Worktrees](/sketchnotes/articles/2026-10-21-worktrees-without-flag-v0163-default-on-migration-codex-cli.png)

**Release context:** v0.163.0 stable (15 October 2026) — PR #54175

---

Managed Git worktrees graduated from opt-in feature flag to always-on default in v0.163.0 stable, released 15 October 2026. For users who were already running the feature experimentally, there are exactly three lines to remove from `config.toml`. For users who haven't tried it yet, the setup cost just dropped to near zero. This article covers both paths.

---

## What Changed in v0.163.0

v0.163.0-alpha.2 (introduced late September 2026) shipped tools for creating and listing managed git worktrees from trusted local projects, gated behind an explicit feature flag. The flag was required to acknowledge that agent-provisioned workspace branches were experimental.

PR #54175 (merged into v0.163.0 stable) retires that flag. Managed worktrees are now unconditionally active for any project marked as trusted in `~/.codex/config.toml`. The flag line no longer has any effect — it is silently ignored rather than rejected, so existing configs with the flag set will not break, but the line is now dead weight.

The trust gate remains. Worktrees are always-on, but only for trusted projects. This is the correct separation: the experimental behaviour was "can agents provision their own worktrees?"; the trust model is "which projects are eligible for agentic workspace changes?" Those are different questions with different governance implications, and v0.163.0 answers them separately.

---

## Before v0.163.0: What You Had to Do

For managed worktrees to work before v0.163.0 stable, a typical `~/.codex/config.toml` needed three things:

```toml
# 1. Global feature flag — now redundant
[features]
managed_git_worktrees = true

# 2. Project trust declaration — still required
[projects]
"/path/to/your/repo" = { trust_level = "trusted" }

# 3. (Optional) project-level worktree config in .codex/config.toml
[worktrees]
base_branch = "main"
```

Line 1 — the `[features]` block with `managed_git_worktrees = true` — is the line that is now redundant.

---

## After v0.163.0: What You Need

```toml
# Trust declaration — required for any project where agents should provision worktrees
[projects]
"/path/to/your/repo" = { trust_level = "trusted" }
```

That is the entire configuration required. The feature flag block can be deleted. If you have multiple projects, the pattern scales cleanly:

```toml
[projects]
"/home/dev/active-monorepo" = { trust_level = "trusted" }
"/home/dev/side-project" = { trust_level = "trusted" }
"/tmp/review-only-clone" = { trust_level = "untrusted" }
```

The `untrusted` entry is not required — projects without an explicit entry default to untrusted — but stating it explicitly documents intent.

---

## What "Trusted" Actually Means

Trust controls whether a project's `.codex/` layer is loaded at all. For a trusted project, Codex CLI will:

- Load `.codex/config.toml` from the repository root and merge it with the global config
- Load `AGENTS.md` from the repository root into the session context
- Load and register any hooks defined in `.codex/hooks/`
- Spawn any MCP servers listed in the project's MCP configuration
- **Allow agents to create and list managed worktrees** (v0.163.0 and later)

For an untrusted project, all of the above is suppressed. A cloned repository with a malicious `.codex/` directory does nothing — hooks don't fire, MCP servers don't start, and agents cannot provision worktrees.[^1]

The implication for the worktrees change: if you are opening a repository you did not write and do not fully control, do not add it to `[projects]` with `trust_level = "trusted"`. The always-on default applies only within the trust boundary you define.

---

## The Workflow Change: Operator-Provisioned vs Agent-Provisioned

Before v0.163.0, managed worktrees required operator configuration. The operator set up the feature, defined the base branch, and the agent consumed worktrees that were already available. The agent was a consumer of infrastructure.

From v0.163.0 onwards, agents on trusted projects can create their own worktrees. The agent is now a provisioner of infrastructure within the scope the operator has granted.

This is architecturally significant. An orchestrating agent that needs to run four parallel subagents no longer needs the operator to manually create four worktrees in advance. The orchestrator creates them as part of the task plan — four worktrees named for the four feature areas, each with its own Codex session, each working in isolation. When done, the orchestrator can list and inspect them before the human reviews and merges.

The Command Centre reflects this. Worktrees provisioned by an agent appear in the project's worktree list alongside manually created ones, labelled with the session that created them.

---

## AGENTS.md Patterns for Worktree-Aware Agents

With always-on managed worktrees, it is worth encoding your worktree policy in `AGENTS.md` so that any agent operating on the project knows the expected patterns:

```markdown
## Worktree Policy

This project uses managed Git worktrees for parallel task isolation.

- Create a new worktree for any task that modifies more than one file
- Name worktrees with the pattern: `feature/<task-slug>` or `fix/<task-slug>`
- Do not merge directly from a worktree — create a PR for review
- Clean up completed worktrees using the `list_managed_worktrees` tool before ending the session
```

This guidance is loaded by every agent operating on the project, so you do not need to repeat worktree instructions in individual prompts.

---

## The Three Things to Remove from Existing Configs

If you were an early adopter of managed worktrees under the flag, your cleanup is:

1. **Delete the `[features]` block** if `managed_git_worktrees = true` is its only entry. If the block contains other flags (e.g., `voice = true`, which also graduated in v0.163.0), remove only the worktrees and voice lines, or delete the block entirely if both were the only entries.

2. **Remove `voice = true`** from the same block — voice also graduated to always-on in v0.163.0, via the same PR #54175.

3. **Verify the `[projects]` trust entry is still present** — the flag removal does not affect trust configuration, but confirm it survived any config cleanup you performed after the v0.163.0 upgrade.

There is no migration to run, no schema to update, and no session restart required. The change is purely subtractive.

---

## Common Pitfall: The Flag Does Not Disable the Feature

If you were using the flag as a safety switch — enabling worktrees for testing and planning to remove the flag to disable them — that pattern no longer works. In v0.163.0, the flag is ignored and worktrees are always active for trusted projects.

If you need to disable managed worktree provisioning for a specific project, set `trust_level = "untrusted"` for that project, or remove it from the `[projects]` table entirely. That is now the correct control.

---

## Summary

| | Before v0.163.0 | v0.163.0 and later |
|---|---|---|
| Feature flag required? | Yes (`managed_git_worktrees = true`) | No — ignored if present |
| Trust configuration required? | Yes | Yes — unchanged |
| Agent provisioning | Available with flag + trust | Available with trust only |
| Operator pre-setup | Required | Optional |
| Voice flag | Separate `voice = true` entry | Also retired; always-on |

v0.163.0 makes managed worktrees a zero-config capability for any project you trust. The governance control that matters — project trust — remains exactly as it was.

---

[^1]: Codex CLI. *Project trust model — trust_level configuration key*. GitHub. <https://github.com/openai/codex/blob/main/AGENTS.md>

[^2]: PR #54175 — managed worktrees and voice flags retired to stable defaults. OpenAI Codex CLI. <https://github.com/openai/codex/releases/tag/rust-v0.163.0>

[^3]: OpenAI. *Codex CLI v0.163.0 Release Notes*. GitHub. <https://github.com/openai/codex/releases/tag/rust-v0.163.0>
