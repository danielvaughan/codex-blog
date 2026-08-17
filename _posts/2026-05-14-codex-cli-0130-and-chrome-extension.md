---
title: "Codex CLI 0.130.0 and Chrome Extension Launch"
description: "Captured: 2026-05-14 Sources:"
date: 2026-05-14T00:00:00+00:00
last_modified_at: 2026-08-17T12:09:31+01:00
type: Technical Article
timestamp: 2026-05-14T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-05-14-codex-cli-0130-and-chrome-extension"
---
![Sketchnote diagram for: Codex CLI 0.130.0 and Chrome Extension Launch](/sketchnotes/articles/2026-05-14-codex-cli-0130-and-chrome-extension.png)

# Codex CLI 0.130.0 and Chrome Extension Launch

Captured: 2026-05-14
Sources: https://developers.openai.com/codex/changelog

## Codex CLI 0.130.0 (May 8, 2026)

Key additions:
- **`codex remote-control`** — new command for remote control capabilities
- **Plugin details enhancements** — improved plugin management and visibility
- **App-server thread pagination** — better handling of long agent sessions
- **AWS Bedrock auth** — login credentials support for AWS Bedrock authentication

## Codex for Chrome (May 7, 2026)

A Chrome extension enabling:
- Parallel cross-tab work from the browser
- User-controlled website access permissions
- Bridges the gap between terminal CLI and web-based workflows

## Codex CLI 0.129.0 (May 7, 2026)

- **Modal Vim editing** in composer — power users can now use Vim keybindings
- **Redesigned resume/fork picker** — cleaner session management
- **Theme-aware status line colours** — better TUI aesthetics
- **Expanded plugin management** — marketplace workflows

## Why This Matters

The `remote-control` command and Chrome extension signal Codex moving beyond terminal-only workflows. Combined with the plugin marketplace maturation, Codex is becoming a platform rather than just a CLI tool.

AWS Bedrock auth is significant for enterprise users who need to run Codex through their own model infrastructure rather than OpenAI's API directly.
