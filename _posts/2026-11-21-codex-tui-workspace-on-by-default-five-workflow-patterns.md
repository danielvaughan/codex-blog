---
title: "Codex TUI Workspace Is Now On by Default: Five Workflow Patterns It Unlocks"
date: 2026-11-21T08:00:00+00:00
last_modified_at: 2026-11-24T07:15:26Z
tags:
  - codex-cli
  - v0.171.0
  - tui
  - workspace
  - agentic
  - subagents
  - image-previews
  - debugging
  - workflow
rating: 4.0
depth: 4
originality: 4
accuracy: 5
substack: 4
copy_edited: true
---

If you opened Codex CLI after updating to v0.171.0 and found your terminal split in two, you are not alone. The TUI workspace panel — previously an opt-in feature requiring a configuration flag — is now on for every user by default. No config change needed, no flag to set. It is just there.

This article explains what the workspace pane actually does, when each of its three capabilities matters, how to collapse it if the layout does not suit you, and five workflow patterns that the workspace makes practical for the first time.

---

## What the Workspace Panel Is

The Codex TUI is a full-screen terminal interface. Before v0.171.0, it was a single pane: prompt composer at the bottom, conversation history above. The workspace adds a persistent side panel on the right that hosts three distinct capabilities.

**1. Searchable document previews**

When Codex reads a file into context — whether you attached it manually or the agent fetched it via a tool call — the workspace panel displays it with inline search (Ctrl-F). Previously, if you wanted to verify what section of a large file the agent had actually seen, you had to open the file separately in another terminal or editor tab. Now it surfaces in-pane alongside the conversation.

This matters most with generated output. If Codex produces a long file and you want to scan it before approving a write operation, the workspace preview lets you do that without leaving the TUI.

**2. Image previews**

Image attachments render inline in the workspace panel as actual images rather than as file-path references. This applies to screenshots you paste into the prompt, images the agent fetches from URLs, and any output images a tool produces.

The practical win is verification. When you paste a screenshot of a UI bug for Codex to fix, you can confirm it has the right image in context — not just a path string — before committing tokens to the task. And when Codex generates or modifies an image, you see the result immediately without opening a separate viewer.

**3. Subagent transcripts**

This is the most substantive capability for teams running agentic workflows. When Codex dispatches a subagent — via `/agents` or through a tool call that spawns a parallel worker — the subagent's session transcript streams into the workspace panel in real time alongside the orchestrator conversation.

Before v0.171.0, subagent activity was largely opaque from within the TUI. You could see that a subagent had been dispatched and that a result had returned, but the intermediate reasoning, tool calls, and partial outputs were invisible during execution. With the workspace enabled, you watch the subagent work: each tool call, each intermediate reasoning step, each file read.

---

## How to Collapse or Adjust the Workspace

If the split layout is not useful for your workflow — perhaps you are working on a small screen, or running Codex in a narrow terminal pane within a tmux layout — you can collapse the workspace.

The workspace panel respects standard TUI resize bindings. Use the focus keys to move between panes and resize with the bracket keys (`[` and `]`) when the pane divider is focused. To collapse it entirely, reduce it to minimum width; on next launch, it will restore to default unless you set the preference in config.

For users who want to suppress the workspace permanently without updating per-session:

```toml
# ~/.codex/config.toml
[tui]
workspace_enabled = false
```

This overrides the new default and returns the single-pane layout.

---

## Five Workflow Patterns the Workspace Unlocks

### 1. Debugging a Subagent That Returned the Wrong Answer

Previously: subagent returns unexpected output → you have no visibility into what went wrong → you re-run with verbose logging and try to correlate output lines with the problem.

With workspace: the subagent transcript streams in real time. You can watch tool calls as they execute and see exactly which call produced the divergence — the wrong file read, the malformed query, the tool that returned an unexpected schema — without instrumenting anything or re-running.

This significantly cuts debugging cycles for multi-agent tasks. The first run is now also the diagnostic run.

### 2. Verifying File Context Before a Destructive Write

Codex sometimes operates on files at scale: batch renames, bulk refactors, schema migrations. Before approving a write, you want to verify the agent has the right version of the file in context, not a stale cached copy from earlier in the session.

With the workspace document preview open, you can inspect the file the agent is working from before you approve the write. The searchable view means you can Ctrl-F for a specific function name or config key to confirm you are looking at the correct version.

### 3. Reviewing a Screenshot-Driven UI Fix

You paste a screenshot of a broken mobile rendering. The agent proposes a CSS fix. Before you apply it, you want to confirm the agent actually received the screenshot you intended — not an earlier attachment still lingering in context.

With inline image preview, the workspace shows the image the agent is working from. One glance confirms you are aligned. If the wrong image is showing, you can re-attach before the agent writes any changes.

### 4. Parallel Research with Live Progress

You dispatch two subagents in parallel: one to fetch and summarise the v0.173.0-alpha.1 release notes, one to scan the GitHub issue tracker for any open regressions against v0.171.0. While they work, you continue a conversation with the orchestrator about which issues to prioritise.

With the workspace panel, both subagent transcripts are visible as they stream. You can track their progress, notice if one stalls, and intervene without waiting for the final consolidated result. The orchestrator conversation and both sub-tasks are all visible simultaneously.

### 5. Document-Driven Code Review

You open a large pull request diff as a document in the workspace. As Codex reviews each section, the document preview stays open in the side panel. You follow along, Ctrl-F-ing to the specific functions the agent is commenting on. The review conversation and the source material are co-located — no tab-switching.

---

## What This Signals About Codex's Direction

The workspace panel closing the gap with IDE panels is not coincidental. VS Code's side panels, JetBrains tool windows, and Cursor's reference panels have established a pattern: the conversation and the artefacts it references should be visible simultaneously.

Codex has historically differentiated itself as a terminal-native tool for developers who live in the CLI. The workspace panel extends that positioning rather than abandoning it. You do not leave the terminal; the terminal gains the reference-pane capability that makes extended agentic sessions practical.

The subagent transcript view in particular addresses the main friction point in multi-agent CLI workflows: observability. As Codex moves toward longer agentic runs — overnight builds, background research sweeps, parallel test execution — the ability to watch sub-tasks without polling external logs becomes a core capability rather than a convenience.

---

## References

- Codex CLI v0.171.0 release notes — github.com/openai/codex/releases/tag/rust-v0.171.0 (2026-11-20)
- Pull requests: #61352, #61888, #61889, #61919, #62157 (TUI workspace enablement by default)
- Related KB article: [Codex CLI v0.171.0 Stable: GPT-6.2 Sol Default, TUI Workspace On, Remote Exec, and MCP Status Line](2026-11-20-codex-cli-v0171-stable-gpt62-sol-default-tui-workspace-remote-exec-mcp-statusline.md)
