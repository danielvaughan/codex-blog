---
title: "The Git Bash Trap: How Codex CLI Silently Picks the Wrong Shell on Windows"
parent: "Articles"
nav_order: 1232
date: 2026-11-13T09:00:00+00:00
last_modified_at: 2026-11-13T03:14:17Z
tags:
  - codex-cli
  - windows
  - wsl
  - git-bash
  - shell
  - mcp
  - bugs
  - tool-calls
  - community
copy_edited: true
---

There is a category of software bug more damaging than a crash: the kind that silently does the wrong thing and lets you discover the consequences hours later. GitHub issue #60793, filed against Codex CLI on Windows, describes exactly this pattern.

When a developer explicitly configures the absolute path to Git Bash as their shell — `C:\Program Files\Git\bin\bash.exe` — Codex CLI ignores it and resolves to WSL bash via `PATH` instead. No warning. No error. The session starts, the agent runs, and tools execute inside the wrong runtime environment. The developer only realises something is wrong when commands produce Linux MINGW64 output where they expected Windows Git Bash behaviour, or when paths, environment variables, and tool availability diverge from what their project expects.

It is a silent misconfiguration that the tool actively hides from you.

## What `exec_command` Is Supposed to Do

Codex CLI's `exec_command` configuration lets developers specify the shell binary used to execute tool calls during an agent session. On Windows, this matters enormously. The Windows developer landscape is fragmented across at least three meaningful shell environments: native `cmd.exe`, PowerShell, Git Bash (a MSYS2-derived Unix-like shell bundled with Git for Windows), and WSL bash (a full Linux kernel via Windows Subsystem for Linux).

Each environment has different behaviour. Path separators differ. Environment variables differ. Tool availability differs. `curl` in Git Bash behaves differently from `curl` in WSL. `git` configured in Git Bash does not necessarily see the same credential store as `git` in WSL. A project that works correctly in one environment can fail silently in another.

The purpose of `exec_command` is to let a developer say: *use this specific shell, with this specific path, so my project's toolchain is consistent*. It is a correctness control, not a preference.

Issue #60793 reports that this control does not work. When `exec_command` is set to an absolute path to Git Bash, Codex CLI falls back to `PATH` resolution — and on machines where WSL is installed, `PATH` often resolves to WSL bash first. The explicit absolute path is disregarded.

## The MINGW64 Environment Signal

The reporter describes the symptom: commands produce output characteristic of a Linux MINGW64 environment. This is a telltale sign of Git Bash being incorrectly invoked from within a WSL context, or of environment variable leakage between the two subsystems. MINGW64 is the MinGW-w64 toolchain environment that Git Bash builds on top of — it should appear in a Git Bash session on Windows, not in WSL.

The confusion arises because WSL and Git Bash share some conventions (both present Unix-like environments) while differing in ways that matter for agentic tool execution (filesystem paths, process isolation, Windows API access). A developer who sets up `exec_command` to use Git Bash specifically may be doing so because their project relies on Windows-native `git credential manager`, Windows path conventions, or tools installed under `C:\Program Files` — none of which are accessible from WSL bash.

When Codex CLI silently routes execution to WSL bash instead, those tools are unavailable. The agent proceeds anyway, potentially producing incorrect output, corrupted file paths, or failed tool calls that surface as cryptic errors rather than clear shell-mismatch diagnostics.

## MCP Server Breakage

The issue compounds when MCP servers are involved. Issue #60723, filed the same week, reports a complete MCP server startup crash on Windows when any project-level MCP server is configured. The crash occurs during TUI bootstrap with a JSON-RPC -32603 internal error.

The relationship between the two issues is not formally confirmed, but the failure mode is consistent: Codex CLI's shell resolution on Windows is unreliable, and MCP server startup depends on the same resolution path. If the shell environment is wrong, the MCP process spawn is wrong, and the MCP server fails before it can respond to its first message.

For teams using MCP to give Codex CLI access to custom tools — file systems, internal APIs, CI/CD integrations — this means MCP is effectively broken on Windows as of v0.167.1. The workaround is to disable project-level MCP configuration, which defeats the purpose of using MCP.

## Why Silent Failure Is the Worst Kind

If `exec_command` pointed to a non-existent path, the correct behaviour would be an immediate error: *"The specified shell binary was not found. Check your exec_command configuration."* The developer would see the problem, fix the path, and move on.

The actual behaviour — silently falling back to PATH resolution — is far more harmful. The agent session starts successfully. Tool calls execute, apparently correctly. The developer may complete an entire work session before noticing that something is wrong, at which point the damage may include committed code with incorrect paths, failed CI runs, or corrupted state in tools that expected Git Bash conventions.

The issue reporter states the fix clearly: honour explicitly supplied absolute executable paths. If a developer provides `C:\Program Files\Git\bin\bash.exe`, use that path. If the binary at that path is not executable, return an error immediately. Do not silently substitute a different shell.

This is not a novel design principle. It is the standard contract for explicit configuration values: explicit beats implicit, and silent fallback should require deliberate opt-in, not be the default.

## The Broader Windows Platform Pattern

Issue #60793 is one of seven Windows-related issues filed against Codex CLI in a 24-hour period in mid-November 2026 — the third major cluster of Windows platform failures in four weeks. The pattern across the cluster is consistent: shell resolution (silently wrong runtime), MCP configuration loading crashes, and WSL integration mismatches.

The cumulative effect is a Windows developer experience that feels structurally undertested. Each individual issue might be dismissed as an edge case. The cluster pattern is harder to dismiss: Git Bash users, WSL users, and MCP users are core Windows developer segments, not fringe configurations. If Codex CLI cannot reliably pick the right shell when you explicitly tell it which one to use, the correctness contract for the entire tool execution layer is broken on Windows.

The fix for #60793 is small — respect the absolute path, error on invalid path, do not consult `PATH` when an explicit path is given. But the signal it sends matters. Until it ships, Windows developers who rely on Git Bash for consistent toolchain execution cannot trust that their `exec_command` configuration does what it says.

---

*References:*

[^1]: GitHub issue #60793 — `exec_command` silently ignores explicit Git Bash path: https://github.com/openai/codex/issues/60793
[^2]: GitHub issue #60723 — MCP server startup crash on Windows (v0.167.1): https://github.com/openai/codex/issues/60723
[^3]: Codex CLI Windows/WSL platform issue cluster, November 2026 (issues #60793, #60723, #60874, #60801, #60600, #60549, #60204)
