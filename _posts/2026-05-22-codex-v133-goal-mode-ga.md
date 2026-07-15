---
title: "Codex CLI v0.133: Goal Mode Goes GA"
description: "Codex CLI v0.133.0 (released 21 May 2026) marks Goal Mode moving from experimental to generally available."
date: 2026-05-22T00:00:00+00:00
last_modified_at: 2026-07-15T08:22:35+01:00
type: Technical Article
timestamp: 2026-05-22T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-05-22-codex-v133-goal-mode-ga"
---
![Sketchnote diagram for: Codex CLI v0.133: Goal Mode Goes GA](/sketchnotes/articles/2026-05-22-codex-v133-goal-mode-ga.png)

# Codex CLI v0.133: Goal Mode Goes GA

*Published: 2026-05-22*

## Summary

Codex CLI v0.133.0 (released 21 May 2026) marks Goal Mode moving from experimental to generally available. This is significant for agentic workflows — Goal Mode allows Codex to drive toward a specific objective for hours or days, with persistence across sessions.

## What Changed

### Goal Mode GA
- Enabled by default with dedicated storage
- Available across app, IDE extension, and CLI
- TUI controls: create, pause, resume, clear
- App-server APIs for programmatic goal management
- Model tools and runtime continuation support

### Permission Profiles
- List APIs for inspecting available profiles
- Inheritance support — compose profiles from base profiles
- Managed `requirements.toml` support
- `codex doctor` (v0.131.0) for diagnostics

### Plugin & Extension Improvements
- Marketplace CLI commands for plugin discovery
- Version-aware sharing and checkout
- Default-enabled plugin hooks
- Extensions can now observe subagent start/stop, tool execution, turn metadata

## Why This Matters for Agentic Workflows

Goal Mode GA means long-running autonomous workflows are now a first-class feature. Combined with subagents, hooks, and MCP integration, Codex can now:

1. Accept a high-level objective (e.g., "migrate all API endpoints to v3")
2. Plan and decompose the work
3. Spawn subagents for parallel execution
4. Persist progress across sessions
5. Resume after interruption

This is the foundation for the "agentic pod" pattern — a team of specialised agents working toward a shared goal.

## Sources

- [Codex GitHub Releases](https://github.com/openai/codex/releases)
- [Codex Changelog](https://developers.openai.com/codex/changelog)
