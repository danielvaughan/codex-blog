---
title: "Codex CLI v0.172.0 Stable: Lazy MCP Startup, Slash-Command Tab Completion, Bedrock Runtime, and Cross-Platform Remote Sessions"
date: 2026-11-25T09:00:00+00:00
last_modified_at: 2026-11-25T10:27:40Z
tags:
  - codex-cli
  - v0.172.0
  - release-notes
  - mcp
  - tui
  - remote
  - bedrock
rating: 3.5
depth: 3
originality: 2
accuracy: 5
substack: 3
copy_edited: true
---

**Release date:** 24 November 2026, 00:03 UTC
**Version:** v0.172.0 stable
**Install:** `npm install -g @openai/codex@0.172.0`
**Full diff:** [rust-v0.171.0...rust-v0.172.0](https://github.com/openai/codex/compare/rust-v0.171.0...rust-v0.172.0)

v0.172.0 is a startup-latency and polish release. Nothing in it changes the model catalogue; the changes that matter are an MCP default that shifts when servers launch, a handful of TUI navigation improvements, and fixes that make resumed and remote sessions keep their sandbox state. All items below are taken from the [official release notes](https://github.com/openai/codex/releases/tag/rust-v0.172.0).

## MCP: Lazy Startup by Default (#62366)

Optional stdio MCP servers that have a cached tool list now start on their **first tool call** rather than at session start. If you configure many servers and use few per session, expect a faster start. The flip side: the first call to a given server pays the launch cost, so a server that fails to start now surfaces its error mid-session rather than at the beginning. Check your servers still launch cleanly before relying on them in unattended `codex exec` runs.

## Approvals: Per-Tool Reviewers (#62360)

You can set an approval reviewer for individual connected app tools in `config.toml`, rather than only for a server as a whole. The release notes do not give the exact key names, so consult the current configuration reference before editing.

## TUI Improvements

- **Tab completion for slash-command arguments** (#62343): `/mcp lo` completes to `/mcp login`.
- **Thread links open in the terminal** (#62332): links in search results and transcripts open saved conversations and nested agents.
- **Task-group navigation** (#62200): `Shift+Up` and `Shift+Down` jump between task groups in the agent command center.
- **Paging long choices** (#62209): `PageUp` and `PageDown` scroll answer choices taller than the answer area.
- **Cloud replies** (#62403): replies finishing after history loads update in place, keeping transcript order.

## Providers

App-server clients can select **Amazon Bedrock Runtime** during setup or sign-in (#62222). Provider errors are now reported as soon as they arrive in a response stream, and they identify which API spend limit was reached (#62329, #62356).

## Resume and Remote Fixes

- `codex exec resume` restores saved sandbox permissions, working directory and workspace roots unless explicitly overridden (#62194). Previously-resumed runs that relied on the old behaviour should be re-checked.
- Remote sessions preserve working directories, workspace roots and writable paths when client and server run on different operating systems (#62375, #62376, #62377).
- Interrupting a parent turn preserves queued replies from child agents for the next turn (#62348).

## Docs and Chores

The bundled `openai-docs` skill now covers migration to GPT-6.2 Astra and Sol, including endpoint choices and tool-calling requirements (#62447). **Code Mode now uses gRPC for all host connections, and the `code_mode_host_grpc` setting is deprecated** (#62476); remove it from configs you maintain.

## Upgrade Checklist

1. `npm install -g @openai/codex@0.172.0`, then `codex --version`.
2. Remove `code_mode_host_grpc` from any config.
3. Start each MCP server you depend on once, to confirm lazy launch works.
4. Re-run any scripted `codex exec resume` flows and confirm the restored sandbox permissions are what you intend.

## Sources

- [Codex CLI release rust-v0.172.0](https://github.com/openai/codex/releases/tag/rust-v0.172.0)
- [Compare rust-v0.171.0...rust-v0.172.0](https://github.com/openai/codex/compare/rust-v0.171.0...rust-v0.172.0)
