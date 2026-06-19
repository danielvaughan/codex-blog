---
title: "Codex CLI v0.130.0: What's New and Why It Matters"
description: "The latest stable release of Codex CLI (v0.130.0, 8 May 2026) brings several features relevant to agentic and enterprise workflows."
date: 2026-05-11T00:00:00+00:00
last_modified_at: 2026-06-19T10:09:35+01:00
tags: [codex-cli, changelog, plugins, mcp, remote-control]
parent: "Articles"
nav_order: 935
---

![Sketchnote diagram for: Codex CLI v0.130.0: What's New and Why It Matters](/sketchnotes/articles/2026-05-11-codex-v0130-whats-new.png)


# Codex CLI v0.130.0: What's New and Why It Matters

Published: 11 May 2026

The latest stable release of Codex CLI (v0.130.0, 8 May 2026) brings several features relevant to agentic and enterprise workflows.

## Plugin Marketplace

Codex now supports installing plugins from GitHub repos, git URLs, local directories, and direct `marketplace.json` URLs. Plugin details display bundled hooks, and sharing exposes link metadata with discoverability controls. This is a significant step toward a composable agent ecosystem.

## Remote Control

The new `codex remote-control` command provides a simpler entrypoint for starting a headless, remotely controllable app-server. Combined with `--full-auto`, this is the recommended pattern for CI/CD and headless deployments.

## MCP Enhancements

- **Namespaced MCP registration** -- cleaner tool organisation across multiple servers
- **Parallel-call opt-in** -- MCP servers can now declare support for parallel tool calls
- **Sandbox-state metadata** -- MCP servers receive information about the current sandbox state

## App-Server Improvements

- **Pagination** for large threads (unloaded/summary/full turn item views)
- **Live config reload** -- no restart needed when config changes
- **AWS Bedrock** auth supports console-login credentials from `aws login` profiles

## Multi-Environment Sessions

`view_image` now resolves files through the selected environment, enabling proper multi-environment workflows.

## What This Means for Agentic Workflows

The plugin marketplace + MCP enhancements + remote control create a foundation for:
1. **Composable agent toolchains** -- install specialised plugins per project
2. **Headless orchestration** -- `remote-control` + `--full-auto` for pipeline integration
3. **Cross-agent interop** -- namespaced MCP makes it cleaner to run multiple MCP servers

Sources:
- [GitHub releases](https://github.com/openai/codex/releases)
- [Official changelog](https://developers.openai.com/codex/changelog)
