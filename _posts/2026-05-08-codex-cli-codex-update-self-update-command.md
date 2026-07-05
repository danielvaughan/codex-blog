---
title: "codex update: Self-Updating the CLI Without Remembering Your Package Manager"
description: "Codex CLI now ships a built-in codex update command that detects the installation method and applies updates automatically."
date: 2026-05-08T00:00:00+00:00
last_modified_at: 2026-07-05T16:13:54+01:00
category: cli
tags: [codex-cli, update, installation, devx]
source:
  - https://developers.openai.com/codex/changelog
  - https://github.com/openai/codex/issues/9274
  - https://github.com/openai/codex/issues/11169
type: Technical Article
timestamp: 2026-05-08T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-05-08-codex-cli-codex-update-self-update-command"
---
![Sketchnote diagram for: codex update: Self-Updating the CLI Without Remembering Your Package Manager](/sketchnotes/articles/2026-05-08-codex-cli-codex-update-self-update-command.png)


Codex CLI now ships a built-in `codex update` command that detects the installation method and applies updates automatically. No more remembering whether you installed via npm, Homebrew, or standalone binary.

## The Problem

Before `codex update`, upgrading depended entirely on how you installed:

```bash
# npm
npm install -g @openai/codex@latest

# Homebrew
brew upgrade openai-codex

# Standalone binary
curl -sL ... | sh
```

This was a frequent source of confusion, especially for teams where different developers used different installation methods. GitHub issues [#9274](https://github.com/openai/codex/issues/9274) and [#11169](https://github.com/openai/codex/issues/11169) tracked the feature request.

## How It Works

```bash
# Check current vs latest version
codex update --check

# Update to latest
codex update
```

The command auto-detects the install source (npm, Homebrew, standalone binary) and runs the appropriate upgrade command. If the installed release supports self-update, the process is seamless.

## When to Use It

- After seeing the "new version available" banner in the TUI status line
- Before testing features from the changelog
- In CI scripts where you want the latest stable release

## Limitations

- Requires the installed release to support self-update (older versions may not)
- On systems with restricted package manager permissions, you may still need `sudo` or equivalent
- The command updates to the latest stable release; to pin a specific version, use the package manager directly

## Key Takeaway

`codex update` is a small quality-of-life improvement, but it removes a real friction point. For teams standardising on Codex CLI, it means "run `codex update`" is the only instruction needed, regardless of how each developer originally installed it.

---

*Sources: [Codex Changelog](https://developers.openai.com/codex/changelog), [Issue #9274](https://github.com/openai/codex/issues/9274), [Issue #11169](https://github.com/openai/codex/issues/11169). Published 2026-05-08.*
