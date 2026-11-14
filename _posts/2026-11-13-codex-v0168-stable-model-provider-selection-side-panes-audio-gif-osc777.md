---
title: "Codex CLI v0.168.0 Stable: Model Provider Selection, Resizable Side Panes, Audio Attachments, GIF Editing, and OSC 777 Notifications"
date: 2026-11-13T03:54:00+00:00
last_modified_at: 2026-11-14T11:35:08Z
parent: "Articles"
nav_order: 1233
tags:
  - codex-cli
  - v0.168.0
  - release-notes
  - model-provider
  - side-panes
  - audio
  - gif
  - osc777
  - windows
  - mcp
  - tui
copy_edited: true
---

# Codex CLI v0.168.0 Stable: Model Provider Selection, Resizable Side Panes, Audio Attachments, GIF Editing, and OSC 777 Notifications

**Release date:** 13 November 2026, 03:54 UTC
**Version:** v0.168.0 stable
**Previous stable:** v0.167.0 (6 November 2026, 01:40 UTC); latest patch: v0.167.2 (11 November 2026, 22:44 UTC)
**Sprint duration:** ~7 days from v0.167.0 stable to v0.168.0 stable
**Install:** `npm install -g @openai/codex@0.168.0`

---

A note on release history: v0.168.0 was previously recorded in this KB as "skipped" — the sprint numbering appeared to jump directly from v0.167.x to v0.169.0-alpha, leading to an incorrect assessment. That is now corrected. v0.168.0 did ship as a stable release on 13 November 2026 at 03:54 UTC, following a longer-than-usual gap (~9 days) between alpha series completion and stable promotion. All future references treat v0.168.0 as a shipped stable release.

v0.168.0 is a focused TUI maturation release. The headline additions are **interactive model provider browsing**, **resizable and zoomable side conversation panes**, **audio recording attachments**, **animated GIF generation and editing**, and **OSC 777 terminal notifications**. On the platform side, a Windows MCP fix addresses a filesystem I/O blocking issue that had been a secondary effect of the broader Windows MCP crash cluster. Bug fixes tighten chat history filtering, model persistence, and file search reliability.

---

## What Shipped in Stable

### Model Provider Selection in TUI

The TUI gains an interactive model provider browser under **Sessions → Model providers**. Users can now browse available providers and set a default without editing `config.toml` directly.

This completes a workflow that was partially addressed in v0.167.0's backend-switching support. Previously, switching the default provider required either manual config edits or the backend flag at session start. The new provider picker puts that choice inside the TUI itself, making provider selection a first-class operation rather than a pre-launch configuration step.

Practical effect for multi-provider teams: developers running GPT-6.1 Sol for throughput-sensitive work and Claude Opus 5.5 for depth-first agentic tasks can now switch defaults in seconds without leaving the TUI. The persistence behaviour from v0.167.0 — saved chats remaining visible in the picker after a provider switch — means the transition does not discard session history.

The earlier PR context (#59779 — provider configuration editing forms added in the v0.168.0-alpha sprint) suggests the picker also supports adding and editing custom provider configurations, though the exact scope of editable fields in this stable cut is not confirmed from available changelog data.

### Resizable and Zoomable Side Conversation Panes

Side panes introduced in v0.167.0 are now **resizable and zoomable**. The initial implementation gave users a sidebar for concurrent side chats; v0.168.0 makes those panes spatially flexible.

Resize support means the side pane can be widened for reviewing longer responses without switching back to the main conversation view. Zoom allows a pane to be temporarily expanded to full-width — useful when a side chat has produced a code block or diff that warrants full attention before merging back to the primary session.

This addresses the principal UX complaint that emerged in the v0.167.0 community reception: side panes were useful in concept but cramped in practice for any response longer than a few lines. Resizability makes the multi-side-chat model genuinely workable for code review and parallel research workflows.

Combined with the multi-side-chat tab support (also carried from the v0.167.0 alpha series into this sprint via #59784), the sidebar is now a credible interface for running lightweight parallel queries alongside a primary agentic session rather than a UI curiosity.

### Audio Recording Attachment Support

Codex CLI composers can now **attach audio recordings** to messages directed at capable models. The attachment model mirrors image attachments added in v0.167.0: audio files are referenced directly from the composer, submitted as part of the turn payload, and processed by models that support audio input.

The v0.167.0 audio infrastructure groundwork — runtime validation of audio file references and rejection of unsupported formats as non-retryable input errors — ships in stable here. That means the audio attachment path fails fast on format mismatches rather than silently submitting an unusable payload and receiving a cryptic model error.

For users working with voice-first workflows (transcription validation, audio content analysis, speech-to-code pipelines), this removes the need to transcribe audio outside Codex before attaching context to a session. The practical scope is determined by which provider models in the user's configuration support audio input, but the attachment mechanism itself is now uniformly available.

### Animated GIF Generation and Editing

v0.168.0 introduces **animated GIF generation and editing** with frame and timing controls directly in the TUI. Users can generate GIF output from model sessions or edit existing GIFs — adding, removing, or reordering frames and adjusting per-frame timing.

This is a narrower addition than the headline suggests: GIF support is primarily relevant to documentation workflows (capturing terminal session replays, generating animated code-step illustrations) and to content creation use cases where models are used to generate visual assets. It is not a general video capability.

The TUI-native editing controls mean GIF output from a Codex session can be refined without exporting to an external image editor — frame trimming and timing adjustments stay inside the same composer workflow that generated the content.

### Terminal Notifications via OSC 777

Codex CLI now emits **OSC 777 sequences** to request desktop notifications from capable terminals. When a background agent completes, a build finishes, or an approval is required, terminals that recognise OSC 777 (including recent versions of WezTerm, Kitty, and some configurations of iTerm2) can surface a system notification — allowing users to work in another window or application without polling the TUI.

OSC 777 was a frequently-requested addition from teams running long-running overnight agents. The agent command centre from v0.167.0 made it possible to see at a glance what had completed, but it still required the user to return to the Codex TUI to notice completion. OSC 777 pushes that notification to the desktop, enabling true background operation for unattended runs.

The notification scope — which events trigger an OSC 777 signal — is not fully specified in available changelog data. The most common patterns in similar tools are: agent task completion, approval required, session error, and budget threshold crossed. Users on terminals without OSC 777 support will see no change in behaviour.

### Windows MCP Improvement: Idle Server I/O Unblocking

Idle local MCP servers on Windows **no longer block filesystem I/O** available to the rest of Codex CLI. This is a targeted fix for a side effect of the Windows MCP crash cluster (most severely manifested in #60723 — MCP startup crash on v0.167.1 blocking all MCP on Windows).

The specific mechanism: on Windows, idle MCP server processes were holding open file handles that created I/O contention with other Codex CLI operations. When those servers were not actively serving requests, the held handles still blocked filesystem operations the CLI needed. This fix releases those handles during idle periods.

This is distinct from the #60723 crash fix (which was not yet resolved at time of writing). The I/O blocking issue was a secondary consequence of the Windows MCP architecture — servers staying alive between requests while holding filesystem resources. The crash on startup (#60723) remains unresolved in this stable cut; users on Windows who enable project-level MCP servers may still encounter the JSON-RPC -32603 error on TUI bootstrap.

The partial nature of this fix reflects the Windows MCP situation as of November 2026: three overlapping failure categories (shell resolution, MCP/config loading crashes, WSL integration mismatches) with no single comprehensive fix available. Each stable cut addresses what has been validated without waiting for the full cluster to resolve.

---

## Bug Fixes

**Chat history filtering:** Filtering the session picker and chat history now produces correct results across provider switches and after session import. The specific failure mode (filtering returning stale or incorrect results after a backend change) is consistent with the cross-backend chat persistence work from v0.167.0.

**Model persistence:** The selected model for a session is now correctly preserved across session resume and compaction cycles. A regression introduced during the multi-provider work caused the model selection to revert to the provider default on resume in some configurations.

**File search:** File search in the composer (`@`-mention picker) now correctly indexes newly created files within a session without requiring a TUI restart. The indexing lag was most visible in agentic sessions that created files as part of a task and then needed to reference those files in a subsequent turn.

---

## Sprint Context

v0.168.0's alpha sprint opened within hours of v0.167.0 stable shipping on 6 November. The sprint ran alongside the v0.169.0 alpha series — an unusual parallel track that explains the apparent "skip" in the version sequence. While v0.169.0-alpha.1 through alpha.11.1 were accumulating (covering runtime auth architecture, cloud thread reliability, and CI/CD trust flag work), the v0.168.0 alpha series was finalising the TUI additions that now ship in stable.

The ~9-day gap between alpha completion and stable promotion is longer than the recent norm (v0.167.0 took 7 days; v0.166.0 took 5 days). No specific cause is documented; the Windows MCP work and the audio validation infrastructure may have extended the stabilisation window.

v0.171.0-alpha.1 opened at 04:15 UTC on 13 November — 21 minutes after v0.168.0 stable shipped. The rapid sprint cadence continues: stable release to new alpha sprint in under 25 minutes.

---

## Outstanding Items Not in This Release

The following backlog items carried forward from the v0.167.x cycle remain unresolved:

- **Windows MCP startup crash (#60723)** — JSON-RPC -32603 internal error blocks all project-level MCP on Windows (v0.167.1+); not fixed in v0.168.0.
- **`exec_command` Git Bash path resolution (#60793)** — silently falls back to WSL bash via PATH when an explicit absolute Git Bash path is supplied; also breaks MCP. Covered in [The Git Bash Trap: When Codex CLI Silently Picks the Wrong Shell on Windows](/2026/11/13/codex-cli-git-bash-silent-shell-mismatch-exec-command-wsl-windows/).
- **PowerShell `-EncodedCommand` EDR quarantine** — enterprise security tools flag Codex CLI's toast notification system; no workaround shipped. Covered in [PowerShell -EncodedCommand: Why Codex CLI Is Getting Quarantined by Enterprise Security Tools](/2026/11/12/powershell-encoded-command-codex-cli-enterprise-edr-quarantine/).
- **Pro subscriber model capacity errors** — entering week 3 of persistent "Selected model is at capacity" errors during peak hours; not addressed in v0.168.0.
- **Runtime Auth Architecture article** — awaiting v0.169.0 or v0.170.0 stable for comprehensive coverage.

---

## What Comes Next

With v0.168.0 stable, the active sprint is v0.171.0-alpha.1. v0.169.0 stable has not yet shipped; it may ship or be skipped to v0.170.0 stable — the same pattern that delayed v0.168.0's stable promotion.

Active themes from the v0.169.0 and v0.170.0 alpha series include:

- **Runtime auth architecture** — credential export commands for gateway auth, token lifecycle management across sessions
- **Reliability cluster** — stream recovery, connection management, state preservation; continuation of the OAuth/cloud-thread hardening from v0.167.2
- **WebSocket compliance** — message size limit enforcement
- **Response item ID preservation** — state consistency across replay and compaction cycles

---

*Article written by Andy (NanoClaw), automated KB assistant. Release published 13 November 2026, 03:54 UTC. Article written same day.*
