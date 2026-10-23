---
title: "Codex CLI v0.162.0 Stable: VS Code Stuck-Prompt Fix, Windows Hardening, and TUI Task Details"
date: 2026-10-20T08:00:00+00:00
last_modified_at: 2026-10-23T18:11:01+01:00
parent: "Articles"
nav_order: 1190
tags:
  - codex-cli
  - v0.162.0
  - release-notes
  - vs-code
  - windows
  - tui
  - remote-control
  - codex-cloud
copy_edited: true
---

# Codex CLI v0.162.0 Stable: VS Code Stuck-Prompt Fix, Windows Hardening, and TUI Task Details

![Sketchnote diagram for: Codex CLI v0.162.0 Stable: VS Code Stuck-Prompt Fix, Windows Hardening, and TUI Task Details](/sketchnotes/articles/2026-10-20-codex-v0162-vscode-stuck-prompt-fix-windows-hardening-tui-improvements.png)

**Release date:** 8 October 2026
**Version:** v0.162.0 stable (followed by v0.162.1 patch, 9 October 2026)
**Previous stable:** v0.161.0 (7 October 2026)

---

Codex CLI v0.162.0 shipped on 8 October 2026, one day after v0.161.0, closing out a focused sprint that centred on a significant VS Code extension reliability failure. The alpha run covered thirteen or more iterations (alpha.1 through alpha.13+), an unusually long sprint for a single stable cycle. A patch release, v0.162.1, followed the next day with additional Windows-specific fixes.

This article covers the changes that shipped in v0.162.0 stable and the v0.162.1 patch.

---

## The VS Code Stuck-Prompt Root Cause

The headline fix in v0.162.0 resolved a VS Code extension regression that silently blocked all subsequent prompts in the extension's send queue.

**Root cause:** A `"undefined"` JSON serialisation error in the extension's outbound message queue. When certain message payloads were serialised, a value that should have resolved to `null` or been omitted instead serialised as the string `"undefined"`. This corrupted the outbound message, and — critically — the queue processing logic stopped on that malformed item rather than skipping it. Every subsequent prompt sent to the agent while the corrupted item sat in the queue was silently discarded.

**Impact window:** 3–8 October 2026. Users running Codex in the VS Code extension during this period would have seen prompts appear to submit normally but receive no agent response. The CLI and web interfaces were unaffected; only the extension's send queue was involved.

**Fix:** The serialisation path was patched to handle `undefined` values correctly, and the queue processing logic was hardened to surface serialisation errors rather than halt silently.

For a deeper look at the architectural implications of the failure — why extension-layer queues are a fragility point and how IDE integration differs from direct CLI use — see [VS Code Extension Reliability Collapse](../2026-10-10-vs-code-extension-reliability-collapse-october-2026-stuck-prompt-bug-codex-cli/).

---

## Windows Hardening

v0.162.0 shipped the most concentrated set of Windows-specific fixes in several release cycles. Four distinct areas were addressed:

### ACL Recovery for Malformed Denial States

Windows Access Control List (ACL) entries can reach a malformed "denial" state during certain error recovery paths in Codex's sandbox, where a deny ACE (Access Control Entry) is written but the corresponding allow ACE is not correctly restored. v0.162.0 adds an ACL recovery routine that detects and corrects these malformed denial states on startup and after task completion.

### Junction Fallback to `mklink /J`

Codex uses NTFS junctions for certain workspace isolation paths on Windows. Where the programmatic junction creation API fails (typically due to elevation requirements in some corporate Windows configurations), v0.162.0 adds a fallback that invokes `mklink /J` via a shell command. This makes managed worktree and sandbox path operations more reliable across Windows enterprise environments with non-standard privilege configurations.

### Daemon Publish File-Lock Retries

The managed daemon uses a file-lock mechanism to coordinate publish state. On Windows, file locks can be held transiently by antivirus scanners and Windows Defender. v0.162.0 adds retry logic around the daemon publish file-lock acquisition, reducing spurious daemon startup failures in environments with active on-access scanning.

### Windows Terminal Shift+Enter Newline Fix

In Windows Terminal, pressing Shift+Enter to insert a newline in the Codex composer would in some configurations submit the current input rather than adding a line break. This was a key binding conflict between Windows Terminal's own input handling and Codex's composer. v0.162.0 corrects the key event handling so Shift+Enter reliably inserts a newline in the Codex TUI regardless of the host terminal's configuration.

---

## TUI: Task Details Repositioned

The fullscreen TUI received a layout change in v0.162.0: task detail metadata — model name, reasoning effort level, and project information — has moved from the bottom status area to the top of the task panel.

Before this change, the model and project context were shown at the bottom of the TUI alongside status indicators, where they competed visually with ephemeral status messages. The repositioning places persistent context at the top (where it is stable and immediately scannable) and keeps the bottom status area for transient information such as processing indicators and error notices.

This is a small change in terms of code, but meaningfully improves readability in long-running agentic sessions where the model and project context matter for understanding what the agent is doing.

---

## Archive Mid-Turn with Warnings

v0.162.0 allows users to archive a session while a turn is in progress. Previously, the archive action was disabled during active processing to avoid data integrity issues. The new behaviour archives mid-turn but displays a warning indicating that the current turn's output may be incomplete in the archived record.

This is primarily useful for long-running agentic tasks where a user wants to archive a session for record-keeping without waiting for a multi-step task to complete.

---

## Server Reasoning Defaults in Fresh Fullscreen Threads

When launching a new fullscreen thread from a non-fullscreen context, Codex now correctly applies the server's default reasoning effort level to the fresh thread. Previously, fresh fullscreen thread launches could default to a reasoning effort level that did not reflect the server's configured default, requiring the user to manually adjust reasoning effort after launch.

---

## Remote Control: Managed Daemon Reuse

The `codex remote-control` command now reuses an already-running managed daemon when auto-start is enabled, rather than spawning a separate daemon instance. This prevents the duplication of daemon processes that could occur when `remote-control` was invoked while an auto-started daemon was already active, and ensures that remote control operations share the same session context as the primary TUI.

---

## Environment Tool Stability: Codex Cloud Persistence Fix

A correctness fix for Codex Cloud environment persistence: the published environment for a Cloud task was not being correctly applied to new Cloud tasks spawned from that session. The environment was being computed correctly but dropped during the task creation handshake.

v0.162.0 corrects the handshake so that the published environment is reliably propagated to new Cloud tasks. This is a silent correctness fix — no user-facing UI change — but it affects anyone using Codex Cloud with environment-specific configuration, such as custom tool sets, model overrides, or project-scoped secrets.

---

## v0.162.1 Patch (9 October 2026)

The day after v0.162.0 stable, a targeted patch release addressed two regressions surfaced during community testing on Windows production environments. The specific fixes were not detailed in the public changelog beyond "Windows-specific hot-fixes," but the rapid turnaround (less than 24 hours post-stable) indicates they were regressions introduced in the v0.162.0 Windows hardening work rather than pre-existing issues.

---

## What Is Not in v0.162.0

Several items that were in active development during the v0.162.0 alpha sprint did not reach stable in this cycle:

- **`/agents` parallel task view** — present in alpha builds but not confirmed stable in v0.162.0
- **Cross-device Cloud Environments** — announced at DevDay (29 September) but not part of the v0.162.0 release
- **OSC 7501 terminal completion states** — landed in v0.163.0

---

## Context: DevDay 2026 Timing

v0.162.0 shipped on 8 October, nine days after OpenAI DevDay 2026 (29 September, San Francisco). The two events are independent — v0.162.0 was driven entirely by the VS Code regression fix sprint, not by DevDay announcements. The DevDay features (Codex Cloud cross-device environments, the `/agents` view, Code Review, and Security Cloud) are tracked in the [DevDay 2026 recap](../2026-10-08-openai-devday-2026-recap-codex-cloud-agents-view-gpt61-sol-code-review-security-cloud/) and shipped across subsequent releases.

---

## Upgrading

```bash
npm update -g @openai/codex
codex --version  # should report v0.162.1
```

VS Code extension users who experienced the stuck-prompt issue during 3–8 October should update the extension alongside the CLI. The extension update is bundled with the v0.162.0+ CLI package; no separate extension marketplace update is required.

---


[^1]: Codex CLI changelog and release notes — v0.162.0 stable, 8 October 2026; v0.162.1 patch, 9 October 2026.
[^2]: VS Code extension stuck-prompt root cause: `"undefined"` JSON serialisation error in outbound send queue, reported via community channels 3–8 October 2026.
[^3]: Windows hardening details: ACL recovery, junction fallback, daemon file-lock retries, Windows Terminal Shift+Enter fix — observed via release notes and community reports.
