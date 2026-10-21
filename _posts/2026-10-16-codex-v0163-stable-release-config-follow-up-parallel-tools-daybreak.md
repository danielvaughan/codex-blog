---
title: "Codex CLI v0.163.0 Stable: /config Command, Follow-Up While Processing, and Parallel Read-Only Tools"
date: 2026-10-16T08:00:00+00:00
last_modified_at: 2026-10-21T11:33:58+01:00
parent: "Articles"
nav_order: 1185
tags:
  - codex-cli
  - v0.163.0
  - release-notes
  - config-command
  - parallel-tools
  - daybreak
  - daemon
  - ux
copy_edited: true
---

# Codex CLI v0.163.0 Stable: /config Command, Follow-Up While Processing, and Parallel Read-Only Tools

![Sketchnote diagram for: Codex CLI v0.163.0 Stable: /config Command, Follow-Up While Processing, and Parallel Read-Only Tools](/sketchnotes/articles/2026-10-16-codex-v0163-stable-release-config-follow-up-parallel-tools-daybreak.png)

**Release date:** 15 October 2026, 21:30 UTC
**Version:** v0.163.0 stable
**Previous stable:** v0.162.1 (9 October 2026)

---

Codex CLI v0.163.0 reached stable on 15 October 2026 after a thirteen-iteration alpha sprint (alpha.1 through alpha.10.2, running 8–15 October). The stable release closes out the sprint and ships several user-facing features focused on session control, configuration ergonomics, and parallel tool execution.

This article covers the new capabilities that shipped in the stable build. For the lower-level infrastructure changes (managed worktrees, OSC 7501 terminal status, gRPC over stdio) that defined the alpha sprint's engineering direction, see [Reading the v0.163.0 Alpha Sprint](../2026-10-11-codex-v0163-managed-worktrees-osc-7501-grpc-stdio-agent-ux-maturity/).

---

## New User-Facing Features

### /config Command

The `/config` slash command is the headline addition in v0.163.0. It provides a searchable, in-session interface for reading and modifying Codex preferences across all configuration categories.

```text
/config
```

Entering `/config` opens a preference browser. You can search by keyword (e.g. "model", "theme", "approval") and toggle or update settings without leaving the session. Changes apply immediately; settings that require a restart surface a notification indicating that the session must be restarted for the change to take effect.

Before v0.163.0, modifying preferences meant exiting to a terminal, editing `~/.codex/config.json` or equivalent, and restarting. The `/config` command removes that friction for the most common configuration changes.

Notably, `/config` replaces the need to remember the exact JSON key names for settings. The UI surfaces all available preferences with descriptions, making it practical for users who are not deeply familiar with the configuration schema.

### Follow-Up While Processing

Codex CLI v0.163.0 adds the ability to send follow-up messages while the agent is still processing a previous response. Previously, the input field was disabled during agent execution; any follow-up required waiting for the current turn to complete.

The new behaviour has two components:

1. **Queued follow-up** — Messages typed while the agent is processing are queued and delivered at the end of the current turn, before the next turn begins.
2. **Interrupt turns** — You can interrupt the current turn mid-execution. A follow-up that materially changes the task (e.g. "stop, actually do X instead") will interrupt the running turn rather than queue behind it.

This mirrors the interaction model introduced in several competing tools and addresses one of the most common friction points in long-running Codex tasks: the inability to course-correct without waiting for the agent to finish its current step.

The interruption mechanism is distinct from `instant_interrupt` (shipped in v0.162.0), which provides an explicit keyboard shortcut for immediate turn cancellation. Follow-up while processing is a softer control: queue a message, let the agent finish, or interrupt if the follow-up warrants it.

### Leader Key (Ctrl+X) and Keymap Command

v0.163.0 introduces a configurable leader key, defaulting to `Ctrl+X`. The leader key prefix is used for keyboard shortcuts that would otherwise conflict with terminal or shell keybindings.

Voice mute, previously bound to a direct shortcut, is reassigned to `F9`. The `/keymap` command allows full remapping of the leader key and associated shortcuts.

For teams with standard terminal configurations, the default `Ctrl+X` leader avoids the most common conflicts. For users on custom terminal setups, `/keymap` provides a path to resolve any remaining conflicts without editing configuration files.

### /daemon Command

The `/daemon` slash command restarts managed background servers — the persistent processes Codex manages for language servers, tool daemons, and other long-running services — with automatic reconnection.

```text
/daemon restart
```

Previously, daemon restarts required either a full Codex session restart or manual intervention outside the session. `/daemon restart` gives a clean recovery path when a background service enters a bad state (e.g. after a dependency update, a crashed tool server, or a network interruption that leaves a server in a stuck state).

This is primarily useful in longer sessions with external tool integrations (e.g. a language server or MCP server that requires periodic restarts during active development).

### Parallel Read-Only Tools

Independent read-only tool calls — skill lookups, memory retrievals, and similar operations that do not write state — now execute in parallel rather than sequentially.

In practice, this affects response latency for turns where the agent consults multiple knowledge sources before composing a response. For example, a turn that requires checking three separate skill definitions or memory entries will issue those lookups concurrently rather than waiting for each to complete before starting the next.

The parallelisation is applied conservatively: only truly independent, read-only operations are eligible. Write operations and operations with ordering dependencies continue to execute sequentially.

### Daybreak Default-On

[Daybreak](../2026-05-12-openai-daybreak-codex-security-enterprise-cyber-defence-platform/), the OpenAI notification and announcement layer that surfaces service notices inside the Codex TUI session, is now enabled by default for eligible accounts.

In previous releases, Daybreak was opt-in via configuration. The v0.163.0 stable flips the default for eligible accounts (typically those on Codex plans with Daybreak access). Users who prefer not to see in-session notices can disable the feature via `/config`.

Daybreak notices appear as dismissible banners in the TUI. They cover service disruptions, security advisories, and product announcements relevant to the Codex platform.

### codex exec API Key Selection

The `codex exec` subcommand — used to run non-interactive Codex tasks from shell scripts and CI pipelines — gains an explicit flag for OpenAI API key selection.

```bash
codex exec --api-key $MY_KEY "run the test suite and fix any failures"
```

Previously, `codex exec` inherited the API key from the environment or the default configuration. The new flag enables multi-key workflows: for example, a CI pipeline that runs tasks under different API keys depending on the task type or billing allocation.

### Image Previews in Transcript

Local and generated images now render as bounded thumbnails in the fullscreen transcript view (PR #54216). Previously, image references appeared as file paths or URLs only; the transcript view required leaving Codex to view the actual image.

The thumbnails are size-bounded to avoid disrupting transcript readability. Clicking or selecting a thumbnail opens the full image.

This is most useful in workflows that involve image generation or visual asset review — for example, iterating on a generated diagram or reviewing screenshots captured during a test run.

### Syntax Theme in Markdown Prose

The active syntax theme's colour palette is now applied to Markdown prose elements in the TUI (PR #54217): headings, emphasis, lists, and links pick up theme colours rather than using monochrome defaults.

For users with custom themes (configured via `/config` or the theme files), this brings prose rendering into visual alignment with code block rendering.

---

## Bug Fixes

The v0.163.0 stable includes fixes for:

- **Thread permission restoration** — permissions on conversation threads could be incorrectly reset under certain session recovery paths
- **Network proxy security checks** — edge cases in proxy configuration validation
- **Side conversation navigation** — navigation state could desync when switching between side conversations
- **Clipboard on Windows/WSL** — clipboard operations failed in some WSL2 configurations
- **Command output collection** — tool output collection was incomplete for certain long-running shell commands

---

## Feature Flags Retired

Two features previously behind opt-in feature flags are now stable defaults in v0.163.0 (PR #54175, merged pre-release):

- **Managed worktrees** — enabled by default for trusted local projects
- **Voice** — available by default (no feature flag required)

If you were enabling these features explicitly in configuration, you can remove those flags; they are now unconditionally active.

---

## What Is Not in v0.163.0

The changelog mentions several areas where work is ongoing but did not reach stable in this cycle:

- The `/agents` parallel task management view (covered in passing in the [DevDay 2026 recap](../2026-10-08-openai-devday-2026-recap-codex-cloud-agents-view-gpt61-sol-code-review-security-cloud/)) does not have a dedicated stable release yet
- Cross-device cloud environments (a pending article topic flagged in the backlog)
- Guardian Subagent from v0.160.0 — still undocumented in a standalone article

---

## Post-Release Commits (Oct 16)

Three commits have landed on main since the v0.163.0 tag, indicating active post-release work:

| Commit | Description |
|--------|-------------|
| `6ae8b8a` | V8 upgrade to 152.2.0 — Array.sort safety guards backport |
| `a45b8cc` | Streaming file replacement in filesystem abstractions — large-file handling groundwork |
| `e790ce2` | Synchronous mutex for remote metadata request tracking — race condition fix in parallel remote-metadata fetches |

The two Adam Perry commits targeting filesystem and concurrency primitives suggest v0.163.1 or v0.164.0 will focus on large-file robustness for agentic tasks.

---

## Upgrading

```bash
npm update -g @openai/codex
codex --version  # should report v0.163.0
```

For teams pinning versions in CI, update the pinned version in your package.json or Dockerfile to `0.163.0`.

---


[^1]: Codex CLI changelog and release notes — v0.163.0 stable, 15 October 2026. Observed via community digests and official release notes.
[^2]: PR #54175 — managed worktrees and voice flags retired to stable defaults.
[^3]: PR #54216 — image previews in fullscreen transcript.
[^4]: PR #54217 — syntax theme applied to Markdown prose.
