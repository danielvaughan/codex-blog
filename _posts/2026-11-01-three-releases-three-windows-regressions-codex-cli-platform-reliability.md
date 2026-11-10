---
title: "Three Releases, Three Windows Regressions: Codex CLI's Persistent Platform Reliability Problem"
date: 2026-11-01T10:00:00+00:00
last_modified_at: 2026-11-10T18:09:56Z
parent: "Articles"
nav_order: 1212
tags:
  - codex-cli
  - windows
  - reliability
  - regression
  - platform
  - v0.164.0
  - v0.165.0
  - v0.166.0
  - community
  - analysis
copy_edited: true
---

# Three Releases, Three Windows Regressions: Codex CLI's Persistent Platform Reliability Problem

![Sketchnote diagram for: Three Releases, Three Windows Regressions: Codex CLI's Persistent Platform Reliability Problem](/sketchnotes/articles/2026-11-01-three-releases-three-windows-regressions-codex-cli-platform-reliability.png)

**Sprint window:** 5 October – 1 November 2026 (28 Days of Shipping)
**Releases examined:** v0.164.0 (Oct 23), v0.165.0 (Oct 27), v0.166.0 (Oct 29)
**Pattern confirmed:** Three consecutive major stable releases. Three significant Windows regressions. All discovered within 48 hours of stable release.

---

The 28 Days of Shipping sprint delivered three stable Codex CLI releases in nine days — v0.164.0, v0.165.0, and v0.166.0. Each carried real progress: enterprise MCP authentication, GCP Vertex AI, GPT-6.1 Terra, OpenTelemetry delivery infrastructure. The sprint demonstrably raised the competitive baseline against Claude Code.

It also delivered three Windows regressions in a row.

This is not a coincidence. It is a pattern — and one that community patience is running out for. This article documents each regression, traces the structural cause, and examines what the Bazel/V8 migration in v0.166.0 does and does not fix.

---

## The Three Regressions

### Release 1: v0.164.0 (23 October 2026) — Data-Integrity P0 and Desktop Crashes

v0.164.0 stable landed at 04:48 UTC on 23 October. Within hours of the release, two Windows-specific P0 reports appeared in the community:

**Agent repeatedly deletes workspace files (data-integrity P0)**
The most serious regression of the three. Windows users reported their Codex agent autonomously deleting workspace files during task execution — not as part of an explicitly approved operation, but as a side effect of sandbox cleanup routines. The issue received three replies before it was triaged, and raised immediate questions about the sandbox permission model on Windows.

**Codex Desktop self-termination (30+ crashes per day)**
Separate from the file deletion issue, Windows users running the Codex Desktop application reported the process self-terminating at a rate of more than thirty times per day — rendering it unusable as a daily driver. This was classified as a daily-driver blocker, not a fringe case.

Both issues emerged alongside cross-platform instability in the same post-release window: a macOS TUI bootstrap `-32603` failure and a Linux 47–49 GB RAM OOM spike during search and task resume. The simultaneous breadth of the outage — all three major platforms affected within 24 hours — pointed to release testing gaps rather than platform-specific causes.

A v0.165.0 alpha backport (`v0.165.0-alpha.1.1`) subsequently shipped Windows MCP sandbox fixes, suggesting the file deletion issue was traced to the sandbox layer.

---

### Release 2: v0.165.0 (27 October 2026) — VS Code 1.143 Launch Failures

v0.165.0 stable shipped at 20:09 UTC on 27 October. By the following day, a new Windows-specific regression had accumulated 35 comments and was classified as blocking:

**Windows VS Code 1.143.1 UI startup failures**
Codex CLI failed to start from within VS Code 1.143.1 on Windows. The failure mode was a launch error in the VS Code extension panel — Codex simply would not initialise when invoked from the IDE. Workaround: launch Codex from Windows Terminal or PowerShell directly.

The failure has a dual cause: the VS Code 1.143.1 release changed the extension host API surface, and Codex CLI's Windows extension integration was not tested against it before v0.165.0 stable. This is a category of regression that a basic pre-release Windows integration test matrix would catch. The fact that it shipped undetected suggests the Windows test suite either does not include VS Code integration scenarios or was not run at the required VS Code version.

---

### Release 3: v0.166.0 (29 October 2026) — Launch Failures Escalate, New Freezes Emerge

v0.166.0 stable shipped at 19:47 UTC on 29 October. Within 48 hours, two new Windows issues had accumulated significant community attention:

**Windows VS Code 1.143.1 launch failure (42 comments, 39 upvotes)**
The VS Code 1.143.1 failure introduced in the v0.165.0 window remained unresolved and continued to accumulate engagement through v0.166.0. The workaround remained: launch from Windows Terminal or PowerShell directly.

**UKUI Wayland paste regression**
A secondary regression affected users running Codex CLI under the UKUI desktop environment on Wayland. Paste operations failed until `WAYLAND_DISPLAY` was unset. Workaround:

```bash
unset WAYLAND_DISPLAY
codex
```

**Windows taskkill.exe / conhost.exe runaway system freeze (Oct 31 — 44 comments, 27 upvotes)**
By 31 October — two days after v0.166.0 stable — a fourth distinct Windows issue had emerged. Users reported `taskkill.exe` and `conhost.exe` processes entering a runaway state that caused system-level freezes on both Windows 10 and Windows 11. With 44 comments and 27 upvotes, it was tracking as the most-engaged new Windows report of the post-sprint period.

---

## Why This Keeps Happening

Three structural factors explain the pattern.

### 1. Windows Is an Integration Surface, Not a First-Class Build Target

Codex CLI is built and tested primarily in Linux-first CI environments. Windows support is real — the binary ships, the sandbox runs — but the integration surface is wide: the npm distribution channel, the VS Code extension API, Windows sandbox process tree management, Windows proxy authentication, and the system-level process management layer that interacts with `conhost.exe` and `taskkill.exe`. Each of these represents a seam where Linux-passing tests can mask Windows-specific failures.

The v0.164.0 file deletion and crash regressions, the v0.165.0 VS Code integration failure, and the v0.166.0 process runaway are all at different seams. There is no single root cause — but there is a common condition: the Windows integration surface is not covered by a test suite that runs at the pace of the sprint.

### 2. Sprint Velocity Compresses the Pre-Release Window

The 28 Days of Shipping sprint ran at 1–2 major stable releases per week for a sustained four-week period. v0.165.0 graduated from alpha to stable in roughly four days; v0.166.0 in roughly two days. At that cadence, the window for platform-specific integration testing is compressed. A test suite that might catch a Windows regression on a two-week cycle may not flag it before a three-day stable graduation.

This is a trade-off, not a mistake. The sprint delivered substantial competitive progress. But sprint velocity and platform reliability are in tension — and Windows bore a disproportionate share of the reliability cost.

### 3. The Process Management Layer Is Fragile on Windows

The `taskkill.exe` / `conhost.exe` runaway — the most recent regression — points to something structurally different from the VS Code integration failures. It affects the operating system's process management layer. Codex CLI spawns sandboxed subprocesses on Windows using Windows-native process tree management. When that layer behaves unexpectedly — runaway processes, failure to terminate subprocess trees cleanly — the result is a system-level event, not just a Codex error. This is the hardest category of Windows regression to catch in CI and the most disruptive for end users.

---

## What the Bazel / V8 Migration Actually Fixes

v0.166.0 shipped a build infrastructure change: Bazel now compiles V8 from source rather than using pre-built blobs. The changelog positions this as improving Windows binary reliability.

That claim is accurate but narrow. Compiling V8 from source removes a class of ABI mismatch failures where a pre-built V8 blob was compiled against different MSVC or Windows SDK versions than the rest of the Codex binary. In v0.165.x and earlier, it was possible for V8 runtime behaviour to diverge on Windows in ways that were non-deterministic and difficult to reproduce outside the specific Windows build environment.

What the Bazel/V8 migration **does**:
- Removes ABI mismatch risk between V8 and the rest of the binary on Windows
- Makes the Windows build reproducible from source — easier to debug, easier to bisect
- Provides a stable foundation for Windows-specific fixes to be applied reliably

What it **does not** fix:
- The VS Code 1.143.1 extension API integration failure (still open as of 1 November)
- The `taskkill.exe` / `conhost.exe` process runaway (still open as of 1 November)
- The UKUI Wayland paste regression (workaround available, no fix shipped)
- The underlying gap in Windows integration test coverage

The Bazel/V8 change is the right foundational move. It will reduce the frequency of hard-to-reproduce Windows binary failures in future releases. It does not retroactively fix the open issues, and it does not add Windows integration testing.

---

## What Windows Users Should Know Now

**Current state (1 November 2026):**

| Issue | Status | Workaround |
|-------|--------|------------|
| VS Code 1.143.1 launch failure | Open, 42 comments | Launch from Windows Terminal or PowerShell |
| taskkill.exe / conhost.exe runaway | Open, 44 comments | Restart Windows Terminal; monitor task manager |
| UKUI Wayland paste regression | Open | `unset WAYLAND_DISPLAY` before launching |
| Desktop self-termination (v0.164.0 era) | Believed resolved in v0.165.x | Upgrade to v0.166.0 |
| File deletion P0 (v0.164.0 era) | Believed resolved via sandbox fix | Upgrade to v0.166.0 |

**Recommended posture for Windows daily drivers:**
- Pin to v0.166.0 stable for now; v0.167.0 alpha introduces fresh risk until a stable graduation
- Launch Codex from Windows Terminal, not VS Code, until the 1.143.1 extension issue is resolved
- Monitor the `taskkill.exe` runaway thread if you run long-horizon agentic workflows on Windows
- The Bazel/V8 change in v0.166.0 improves long-term binary stability; the current sprint releases have been the worst period for Windows users in the tool's history

---

## The Bigger Question

Three consecutive stable releases with significant Windows regressions discovered within 48 hours of graduation are a process signal. It suggests the stable graduation gate does not include a Windows integration smoke test — or if it does, that smoke test is not covering the surfaces that break.

The community's patience metric is visible: the `taskkill.exe` runaway report reached 44 comments and 27 upvotes in under 48 hours on a topic most VS Code users had already filtered out. Windows-specific reliability has become a trust category of its own, separate from feature quality.

The Bazel/V8 migration gives OpenAI the foundation to address this systematically. Whether that foundation gets built on — with Windows integration testing, VS Code version matrices, and explicit Windows reliability KPIs — will determine whether "three releases, three regressions" describes a sprint-specific anomaly or a structural commitment gap.

---

*Companion articles:*
- *[Codex CLI v0.164.0 Stable: /presets, Image Previews, Enterprise MCP Auth, and Bedrock Default-On](/2026/10/23/codex-v0164-stable-release-presets-image-previews-enterprise-mcp-bedrock/)*
- *[Codex CLI v0.165.0 Stable: GCP Vertex AI, GPT-6.1 Terra, and Memory v2](/2026/10/28/codex-v0165-stable-gcp-vertex-gpt61-terra-memory-v2/)*
- *[Codex CLI v0.166.0 Stable: codex queue --resume, Keyboard Transcript Navigation, and GPT-6.1 Terra](/2026/10/31/codex-v0166-stable-queue-resume-keyboard-transcript-terra/)*
- *[28 Days of Shipping: The Final Scorecard](/2026/11/01/28-days-of-shipping-end-of-sprint-scorecard-openai-codex/)*
