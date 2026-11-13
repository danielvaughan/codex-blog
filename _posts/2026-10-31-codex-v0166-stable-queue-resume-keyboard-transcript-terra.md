---
title: "Codex CLI v0.166.0 Stable: codex queue --resume, Keyboard Transcript Navigation, and GPT-6.1 Terra"
date: 2026-10-31T09:00:00+00:00
last_modified_at: 2026-11-13T07:05:41Z
tags:
  - codex-cli
  - v0.166.0
  - release-notes
  - gpt-61-terra
  - session-management
  - async
  - enterprise
  - mcp
copy_edited: true
---

# Codex CLI v0.166.0 Stable: codex queue --resume, Keyboard Transcript Navigation, and GPT-6.1 Terra

![Sketchnote diagram for: Codex CLI v0.166.0 Stable: codex queue --resume, Keyboard Transcript Navigation, and GPT-6.1 Terra](/sketchnotes/articles/2026-10-31-codex-v0166-stable-queue-resume-keyboard-transcript-terra.png)

**Release date:** 29 October 2026, 19:47 UTC
**Version:** v0.166.0 stable
**Previous stable:** v0.165.0 (27 October 2026, 20:09 UTC)
**Sprint duration:** 2 days, 18 hours (alpha.1 to stable — 5 alpha releases)

---

Codex CLI v0.166.0 reached stable on 29 October 2026, just under two days after v0.165.0. It is the fastest stable cadence since the 28 Days of Shipping sprint began. The release ships seven headline changes: a new `codex queue --resume` command for unloaded threads, full keyboard navigation for transcript selection, GPT-6.1 Terra in the model catalogue, enhanced safety-block explanations, `codex doctor --network-only`, consistent Markdown rendering in async UI elements, and a Bazel build migration to source-built V8 with native Windows support. For enterprise users, the sprint also delivers per-chat MCP connection leases and GCP Vertex AI in the TUI onboarding wizard.

For detailed coverage of the Enterprise OpenTelemetry work that dominated the alpha sprint, see [Codex CLI v0.166.0 Alpha Sprint: Enterprise OpenTelemetry, PEM Client Identities, and `codex doctor --network-only`](/2026/10/28/codex-v0166-enterprise-otel-sprint-managed-config-pem-identities-network-diagnostics/).

---

## What Shipped in Stable

### `codex queue --resume` for Unloaded Threads

The most operationally significant command in v0.166.0 is `codex queue --resume`. It addresses a workflow gap that has existed since the queue system was introduced: threads that have accumulated in the queue but were never loaded into memory could not be resumed without first opening them interactively.

```bash
codex queue --resume
```

The command lists all paused or queued threads that are not currently in memory and resumes them in order. For teams running long-horizon agentic workflows where tasks accumulate faster than they are processed — overnight batch jobs, CI-triggered analysis runs, review pipelines — this removes the need to manually open each queued thread before resuming.

The practical effect is a step towards fully asynchronous session management. A workflow can now queue multiple tasks, close the terminal, return the next morning, and resume everything with a single command without manually loading each thread first.

### Keyboard-Driven Transcript Selection

v0.166.0 adds full keyboard navigation for transcript content selection. Three motion types are now supported:

| Motion | Keys | Behaviour |
|--------|------|-----------|
| Word | `w` (forward), `b` (backward) | Select word by word |
| Line | `j` (down), `k` (up) | Select line by line |
| Page | `PgDn`, `PgUp` | Jump by screen-height chunks |

Previously, selecting transcript content for copying required mouse interaction or scrolling through terminal output. The keyboard-native approach matters most for developers who use Codex CLI inside tmux, Zellij, or other multiplexers where mouse selection behaviour is unreliable or disabled. It also makes the CLI more accessible in environments without pointer devices, such as SSH sessions into remote development machines.

This completes the terminal-native UX story that began with the fullscreen transcript refresh in v0.165.x: scrollable history, pinned composer, and now full keyboard selection.

### GPT-6.1 Terra in the Model Catalogue

GPT-6.1 Terra is now available in the Codex CLI model catalogue. Terra replaces Sol as the premium ultrafast option in the model hierarchy.

Select it interactively:

```text
/model gpt-6.1-terra
```

Or set as default in `~/.codex/config.json`:

```json
{
  "model": "gpt-6.1-terra"
}
```

Terra is relevant to the ongoing GPT-6.1 Sol AGENTS.md compliance regression that has been open for 21 days as of v0.166.0 stable. The Sol regression — where the model acknowledges write constraints in AGENTS.md, then re-violates them one to three turns later — is a model-specific behavioural issue. Terra's improved instruction-following is expected to reduce the re-violation rate, though the Sol regression itself remains unacknowledged by OpenAI.

Teams already running Guardian v2 as a constraint enforcement layer will find Terra additive: Guardian catches violations regardless of model, and Terra reduces the rate at which Guardian has work to do.

### Enhanced Safety-Block Explanations

When Codex blocks a request on safety grounds, the user now receives an explanation and a set of selectable safer alternatives rather than a terminal rejection. This resolves a longstanding UX dead end where a safety block provided no path forward.

The change lands in stable from alpha.1 (PRs #57363/#57365). The practical impact is most visible in exploratory workflows where the user is probing the boundary of what is permissible: instead of receiving a flat refusal and having to reformulate blindly, the block now surfaces what alternative framing would succeed.

### `codex doctor --network-only`

`codex doctor` gained a new sub-flag in the alpha sprint: `--network-only`. It runs only the network diagnostics portion of the doctor suite, skipping environment, configuration, and model checks.

```bash
codex doctor --network-only
```

The flag is most useful when diagnosing MCP connection failures, proxy authentication issues, or Vertex AI connectivity problems where full doctor output adds noise. Running the full suite takes several seconds on slow networks; `--network-only` returns results in under a second for most environments.

For a complete reference of the `codex doctor` command and all sub-flags, see the CLI commands reference.

### Markdown Rendering in Async UI Elements

Async UI elements — questions, choices, answers, and replies in the queue and async workflow surfaces — now render Markdown rather than displaying raw text. Before v0.166.0, formatted output in async contexts appeared as literal asterisks, backticks, and heading hashes, which made responses with code snippets or structured lists difficult to read.

This is a quality-of-life fix for anyone using Codex in async or batch modes where formatted output is common.

### Bazel Build with Source-Built V8 and Native Windows Support

The build infrastructure underlying the Codex CLI binary has changed significantly in v0.166.0: Bazel now compiles V8 from source rather than using pre-built blobs. The primary effect is improved Windows binary reliability.

Three consecutive major stable releases (v0.164.0, v0.165.0, v0.166.0) have shipped with Windows regressions discovered within hours of stable release. v0.166.0 itself landed a Windows VS Code 1.143.1 launch failure within 24 hours and a UKUI Wayland paste regression (workaround: `unset WAYLAND_DISPLAY`). The Bazel/V8 migration addresses the underlying build variability for Windows binaries rather than fixing the specific regressions, and is the foundational change needed before Windows-specific fixes can be applied reliably.

---

## Enterprise Changes

### Per-Chat MCP Connection Leases

From alpha.5 (PR #57978), each conversation now holds its own lease on shared MCP connections. Previously, multiple parallel conversations could share connection state on MCP servers, creating a risk of state bleed between sessions running against the same MCP endpoint.

Per-chat leases mean that session A's MCP state — tool calls in flight, cached responses, authentication tokens — is isolated from session B's, even when both sessions connect to the same MCP server. This is a prerequisite for safe multi-agent MCP deployments where several Codex sessions coordinate through shared MCP tooling.

### GCP Vertex AI in TUI Onboarding Wizard

The TUI onboarding wizard, which walks new users through provider setup at first launch, now includes a GCP Vertex AI path alongside the existing AWS Bedrock wizard (added in v0.164.0).

A user's first Codex CLI session now presents three provider paths:
- OpenAI (default)
- AWS Bedrock
- GCP Vertex AI

Multi-cloud is now a first-class onboarding experience rather than an advanced configuration option reached via documentation.

---

## Known Issues at Stable Release

Two regressions were reported within 24 hours of stable landing:

**Windows VS Code 1.143.1 launch failure** — Codex CLI fails to start from within VS Code 1.143.1 on Windows. As of 30 October, the issue has 42 comments and 39 upvotes and is under investigation. Workaround: launch Codex from Windows Terminal or PowerShell directly.

**UKUI Wayland paste regression** — Paste operations fail in the UKUI desktop environment on Wayland. Community workaround: `unset WAYLAND_DISPLAY` before launching Codex CLI.

These follow the same pattern as v0.164.0 (Windows proxy auth) and v0.165.0 (Windows VS Code 1.143). The Bazel/V8 migration is expected to reduce the frequency of platform-specific regressions in future releases.

---

## Upgrade

```bash
npm install -g @openai/codex@latest
```

Verify:

```bash
codex --version
# Expected: 0.166.0
```

---

## Summary

| Feature | Impact |
|---------|--------|
| `codex queue --resume` | Resume all queued unloaded threads in one command |
| Keyboard transcript selection | Word, line, page motions — works in tmux/SSH |
| GPT-6.1 Terra | Improved AGENTS.md compliance vs Sol; replaces Sol as ultrafast default |
| Safety-block explanations | Shows safer alternatives instead of flat rejection |
| `codex doctor --network-only` | Fast network diagnostics without full doctor suite |
| Async Markdown rendering | Formatted output in queue and async UI elements |
| Source-built V8 (Bazel) | Improved Windows binary reliability baseline |
| Per-chat MCP leases | Isolates MCP state per conversation — safe for multi-agent |
| GCP Vertex AI in wizard | Three-provider onboarding at first launch |

v0.166.0 is a quality-of-life and infrastructure release built on the observability and multi-cloud foundations laid in v0.165.0. The queue resume command and per-chat MCP leases are the most operationally significant additions for teams running production agentic workflows.

---

*Published: 31 October 2026*
*Research date: 31 October 2026*
*Sources: OpenAI Codex CLI GitHub releases, changelog, PR history, community threads*
*Tags: v0.166.0, stable-release, queue-resume, keyboard-navigation, gpt-6-1-terra, mcp, enterprise*
