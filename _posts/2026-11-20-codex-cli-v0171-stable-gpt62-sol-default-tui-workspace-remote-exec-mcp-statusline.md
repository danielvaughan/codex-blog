---
title: "Codex CLI v0.171.0 Stable: GPT-6.2 Sol Default, TUI Workspace On, Remote Exec, and MCP Status Line"
date: 2026-11-20T09:00:00+00:00
last_modified_at: 2026-11-25T01:04:51Z
tags:
  - codex-cli
  - v0.171.0
  - release-notes
  - gpt-6
  - tui
  - mcp
  - remote-exec
  - model-selection
  - workspace
rating: 4.5
depth: 4
originality: 3
accuracy: 5
substack: 4
copy_edited: true
---

![Sketchnote diagram for: Codex CLI v0.171.0 Stable: GPT-6.2 Sol Default, TUI Workspace On, Remote Exec, and MCP Status Line](/sketchnotes/articles/2026-11-20-codex-cli-v0171-stable-gpt62-sol-default-tui-workspace-remote-exec-mcp-statusline.png)

**Release date:** 20 November 2026, 00:44 UTC
**Version:** v0.171.0 stable
**Previous stable:** v0.170.0 (18 November 2026)
**Sprint duration:** Absorbed v0.171.0 and v0.172.0 alpha tracks simultaneously; 13 alphas before promotion
**Install:** `npm install -g @openai/codex@0.171.0`

---

v0.171.0 landed at 00:44 UTC on 20 November, two days after v0.170.0, closing an unusually compressed sprint. Where v0.170.0 delivered WebRTC voice and OpenTelemetry hardening, v0.171.0 shifts focus to the model layer and the TUI surface: GPT-6.2 Sol becomes the new default selection, the workspace panel is enabled for all users, and two new remote control primitives arrive that extend Codex beyond the laptop.

Within two hours of the stable tag, v0.173.0-alpha.1 opened — the dual-track alpha pattern continues at pace.

---

## What Changed

### 1. GPT-6.2 Sol and Astra Join the Model Catalogue; Sol Becomes Default (#62106)

The most consequential change in v0.171.0 is a model default switch. GPT-6.2 Sol replaces the previous default and joins GPT-6.2 Astra in the model catalogue. This is the first model default change since v0.168.0.

The timing is deliberate. A quality regression thread on community.openai.com — "Codex is rapidly degrading — please take this seriously" — had accumulated significant engagement by mid-November, with reports of ~67% task failure rates and 66–75% of token budgets consumed correcting Codex's own errors in affected workflows. The pattern appeared to track the GPT-5.x model series. Setting Sol as the new default is OpenAI's clearest signal that the model layer was the root cause.

For most users, the change is automatic: existing sessions that did not pin a model name will switch to Sol on the next session start. Users who pinned a model explicitly in `~/.codex/config.toml` are unaffected.

To verify which model your session is using:

```bash
codex --model gpt-6.2-sol  # explicit override
# or check your config
cat ~/.codex/config.toml | grep model
```

GPT-6.2 Astra is also available in the catalogue for users who want to evaluate it directly:

```bash
codex --model gpt-6.2-astra
```

The community response metric to watch: if the regression thread quiets over the next one to two weeks following the Sol default, it confirms model quality was the root cause. If reports persist, the root cause is deeper.

---

### 2. TUI Workspace Enabled by Default (#61352, #61888, #61889, #61919, #62157)

The TUI workspace panel — introduced in earlier alphas — is now on for all users without requiring feature flag configuration. The workspace provides a persistent side panel alongside the main conversation with three capabilities:

- **Searchable document previews** — files open in the workspace can be searched in-line without leaving the TUI
- **Image previews** — image attachments render inline in the panel rather than as file references
- **Subagent transcripts** — when Codex spawns subagents, their session transcripts appear in the workspace panel alongside the orchestrator conversation

The subagent transcript view is the most significant of the three for agentic workflows. Previously, subagent activity was largely opaque from within the TUI: you could see that a subagent had been dispatched and that a result had returned, but the intermediate reasoning, tool calls, and partial outputs were not visible in real time. With the workspace enabled, those transcripts stream in as the subagent works.

This matters for debugging. When a subagent returns an unexpected result, the transcript in the workspace panel lets you trace which tool call produced the divergence without reaching for external logging.

The workspace panel is enabled by default from v0.171.0. No configuration change is needed for existing installations — the panel appears on next launch.

---

### 3. Configurable Undo/Redo Shortcuts in the Prompt Composer (#61954)

The non-Vim prompt composer gains configurable undo/redo keyboard shortcuts in v0.171.0. The change also extends undo/redo coverage to two specific actions that previously had no undo path:

- **Attachment removal** — if you detach a file or image from a prompt and immediately want it back, undo now restores it
- **Large paste restoration** — pasting a substantial block of text and undoing it now restores the paste in full rather than partially

These are small-surface quality improvements but address friction points that occur frequently in practice. Accidentally removing an attachment mid-prompt, or pasting context into the wrong location, previously required navigating back to the file or retyping the paste source.

Shortcuts are configured in `~/.codex/config.toml` under the `[keybindings]` section. The release notes do not publish a default keybinding list; check `codex --help keybindings` for the current defaults on your installation.

---

### 4. Remote Session Exec: `codex exec --remote <ADDR>` (#61707)

v0.171.0 introduces `codex exec --remote <ADDR>` — a new flag that starts a new noninteractive thread on an existing Codex app server and runs a turn there, using the server's configuration, credentials, and filesystem.

The use case: Codex app servers running on a remote host (a CI runner, a cloud VM, a shared development machine) are now addressable from a local terminal. You provide the WebSocket address of the existing app server and `--remote` creates a new thread there and runs the prompt, without spinning up a new local agent process.

```bash
codex exec --remote wss://10.0.1.42:3000 "run the full test suite and summarise failures"
```

This is the noninteractive complement to the `/rc` and `/remote` TUI commands (below). Where those commands pair a TUI device for interactive remote control, `codex exec --remote` sends a one-shot instruction to a running server without opening an interactive session.

The practical application in multi-agent builds: an orchestrator process can dispatch work to a fleet of Codex app servers by address, without each dispatch requiring a new session startup handshake. To continue a previous session, capture the thread UUID from the first run and use `codex exec resume --remote <ADDR> <UUID>` for follow-up turns.

---

### 5. `/rc` and `/remote` Slash Commands for Device Pairing (#62121)

Two new slash commands arrive in the TUI for managing remote control and device pairing:

- `/rc` — toggle local remote control mode; when active, the TUI accepts inputs from paired remote devices
- `/remote` — display the current pairing code for connecting a remote display or input device to the running session

The pairing mechanism connects to the Agent Host Protocol pattern emerging in the broader ecosystem: GitHub Engineering published "Building the Agent Host Protocol for sessions that continue across devices" in Seroter #913 on 19 November, describing an open protocol for session persistence across devices mid-task. Codex's `/remote` pairing codes are a TUI-native implementation of a similar concept — sessions that survive a device handoff.

The concrete workflow: start a Codex session on a laptop, run `/remote` to generate a pairing code, enter that code on a second device (another terminal, a mobile client, a monitoring dashboard), and the session continues on the second device without losing context.

---

### 6. Custom MCP Status Line via `/statusline` (#61979, #61988, #62037)

When the `tui_extensions_statusline` feature flag is enabled, MCP servers can now author a custom line in the TUI status bar. The `/statusline` slash command controls this from within a session.

The intent is to let MCP extensions surface live state directly in the terminal interface. An MCP server managing a build pipeline could display the current build status. A database MCP could show active connection count. A cost-tracking MCP could display cumulative token spend for the session.

To enable:

```toml
# ~/.codex/config.toml
[features]
tui_extensions_statusline = true
```

The status line is authored by the MCP server on each update cycle; the TUI re-renders it without blocking the conversation.

---

### 7. Bug Fixes

Four targeted fixes ship in v0.171.0:

**Draft preservation on session switch** — switching between sessions previously discarded in-progress drafts in the prompt composer. Drafts now persist across session switches and are restored when you return to the session.

**Permission profiles intact on working directory changes** — changing the working directory mid-session could cause permission profiles to reset to defaults, forcing manual re-configuration. Permission profiles now follow the session rather than the working directory.

**Late completion events no longer cancel approvals** — a timing race where a late-arriving completion event would cancel a pending approval dialog has been resolved. This manifested as approvals that appeared to dismiss themselves unexpectedly in multi-turn sessions.

**Enterprise MCP sign-in for providers without refresh tokens** — MCP providers that use short-lived access tokens without a refresh token mechanism could not complete sign-in flows in enterprise environments where token lifetimes are short. The sign-in flow now handles the token-only case correctly.

---

## What Comes Next: v0.172.0 and v0.173.0

v0.172.0-alpha.2 was running in parallel with v0.171.0 stabilisation (last seen at alpha.2 on 19 November). With v0.171.0 now stable, that sprint may either continue toward its own stable or be absorbed — the pattern has varied across recent cycles.

v0.173.0-alpha.1 opened at 02:50 UTC on 20 November, 126 minutes after v0.171.0 stable. This is consistent with the last two cycles: a new alpha sprint starts within hours of a stable promotion, maintaining continuous forward momentum.

---

## Context: Desktop Build 26.1117.31558 Crash Wave

The same day v0.171.0 CLI shipped, Codex Desktop build 26.1117.31558 introduced a cluster of startup crashes on both platforms:

- **#62703** — macOS ARM64 crashes on startup (EXC_BREAKPOINT/SIGTRAP) on Apple Silicon
- **#62700** — macOS local skills disappear from slash menu
- **#62698** — Windows kernel crashes on startup and during task interruptions
- **#62701** — Windows `cyber_policy` interruptions blocking legitimate Unreal Engine code reviews

These are Desktop-specific issues; the CLI v0.171.0 release is unaffected. A rapid patch to the Desktop build is likely given the breadth of platform coverage. If you are on macOS Apple Silicon or Windows and experience startup crashes after updating Desktop, the workaround is to roll back to the previous Desktop build until a patch is released.

---

## Install

```bash
npm install -g @openai/codex@0.171.0
```

Verify:

```bash
codex --version
# 0.171.0
```

---

*Full technical reference → [Codex CLI v0.171.0 Stable](https://danielvaughan.com/codex-resources/articles/codex-cli-v0171-stable-gpt62-sol-default-tui-workspace-remote-exec-mcp-statusline/)*
