---
title: "Codex CLI v0.166.0-alpha.1: GPT-6.1-Terra, Safety Recourse, and Multi-Process Session Reliability"
date: 2026-10-27T11:00:00+00:00
last_modified_at: 2026-10-28T10:24:08Z
parent: "Articles"
nav_order: 1206
tags:
  - codex-cli
  - v0.166.0
  - gpt-6-1-terra
  - safety
  - tui
  - session-reliability
  - release-notes
  - sprint
copy_edited: true
---

# Codex CLI v0.166.0-alpha.1: GPT-6.1-Terra, Safety Recourse, and Multi-Process Session Reliability

![Sketchnote diagram for: Codex CLI v0.166.0-alpha.1: GPT-6.1-Terra, Safety Recourse, and Multi-Process Session Reliability](/sketchnotes/articles/2026-10-27-codex-v0166-alpha1-gpt61-terra-safety-recourse-session-reliability.png)

**Release track:** v0.166.0-alpha.1 (27 October 2026, 01:33 UTC)
**Previous alpha track:** v0.165.0-alpha.3 (26 October 2026, 21:54 UTC)
**Current stable:** v0.164.0 (23 October 2026, 04:48 UTC)
**Sprint theme:** Safety UX, model tier expansion, multi-process session reliability
**Key engineers:** Eric Traut (safety recourse), Dmitriy Kovalenko (session reliability), Anthony Tafoya (Rust toolchain)

---

Four releases landed in a 28-hour window between 26 and 27 October. Then, at 01:33 UTC on 27 October, a new branch opened: v0.166.0-alpha.1. The 0.165.x sprint — observability, tracing, Guardian v2 — is not yet finished, but the 0.166.x branch signals where OpenAI is heading next.

Three themes define the new sprint from its opening commit: a new model tier (GPT-6.1-Terra), a shift in how safety policy blocks are presented to users, and a set of low-level fixes to how concurrent Codex processes coordinate around shared session state.

---

## The Sprint Context

The 28 Days of Shipping sprint is on day 23 of 28. The sprint ends 1 November 2026.

The v0.165.x observability sprint gave developers tools to see what Codex was doing: turn-level tracing, streaming tracing, tool-dispatch tracing, and Guardian v2 in full conversation mode. That work remains in alpha. v0.165.0 stable has not shipped yet.

Opening v0.166.0-alpha.1 while v0.165.x is still running is not unusual for Codex CLI's release cadence. The two branches will continue in parallel until v0.165.0 ships stable and v0.166.x absorbs the graduating features.

---

## GPT-6.1-Terra: A New Model Tier

The most significant single change in v0.166.0-alpha.1 is the addition of GPT-6.1-Terra to the Codex model catalogue.

Terra is the next step above Sol in the GPT-6.1 family. The PR (`#57337`, backported into the 0.166 branch) adds Terra support to the model selection system and introduces a `features.simplified_selection_hints` flag that adjusts how model capabilities are surfaced in the TUI.

The naming pattern in the GPT-6.1 family follows a progression from lighter to heavier models. Sol (released in the v0.165.x window) brought improved instruction-following over the previous generation. Terra extends that further — based on the PR description, expect stronger reasoning with a corresponding increase in token cost.

The timing matters. The GPT-6.1 Sol AGENTS.md compliance regression has been open for 19 days without an official fix from OpenAI. Terra's arrival in alpha does not close that regression — the regression is a behavioural issue with Sol's instruction-following, not a missing model. But it does give teams a migration path: if Sol's compliance behaviour remains unreliable for AGENTS.md write-constraints, Terra offers an alternative before any model-level patch ships.

For teams running Guardian v2 as an enforcement layer (per the v0.165.0 observability sprint), the arrival of Terra is additive. Guardian v2 detects constraint violations at the tool-dispatch layer regardless of which model is running. Adding Terra to the catalogue gives those teams a model to test that may reduce the frequency of Guardian interventions in the first place.

---

## Safety Recourse: From Dead Ends to Guided Paths

Two pull requests from Eric Traut address one of the more frustrating moments in agentic coding sessions: hitting a safety policy block with nowhere to go.

Prior to v0.166.0-alpha.1, a safety block in the Codex TUI was a stop signal. The block fired, an explanation was shown, and the user was left to either rephrase manually, start a new session, or give up on the task.

**PR #57365 — Offer safer rewrites after safety blocks in TUI**

When a safety policy block fires, Codex now presents selectable follow-up options alongside the block explanation. These are model-generated safer phrasings of the original request — alternatives that accomplish a compatible goal without triggering the policy.

The user sees the block explanation and a numbered list of suggested rewrites. Selecting one continues the session from that point. No manual rephrasing required.

**PR #57363 — Explain safety policy blocks with turn-scoped recourse**

This companion PR adds structured moderation metadata handling to the block explanation layer. The explanation is rendered in-TUI with the blocking policy identified and the turn-scoped context preserved. The 'turn-scoped' detail matters: the recourse options in PR #57365 are generated with knowledge of what was being attempted in the current turn, not as generic suggestions.

Together, the two PRs shift safety blocks from terminal events to recoverable friction. For agentic sessions running long multi-step tasks, this is a meaningful quality-of-life change. A policy block in step 14 of a 20-step workflow no longer necessarily means restarting from scratch.

---

## Multi-Process Session Reliability

Two pull requests from Dmitriy Kovalenko address a class of session corruption bugs that emerge when multiple Codex processes access shared session state concurrently.

This scenario arises more than it might seem. Developers frequently run Codex in multiple terminal windows, in split-pane layouts, or alongside background agent tasks. When two processes attempt to write session index data at the same time, the results have historically been non-deterministic.

**PR #57360 — Bound waits for thread writer coordination lock**

The thread writer coordination lock now has explicit timing parameters: retry every 50 ms, timeout after 30 seconds. Previously, wait behaviour was unbounded. Under high contention, a process could wait indefinitely for a lock that had been abandoned by a crashed or hung peer process.

The 30-second timeout gives stuck processes a defined exit path. After 30 seconds without acquiring the lock, the process surfaces an error rather than hanging silently.

**PR #57358 — Serialise session index mutations across processes**

Session index mutations are now guarded by a persistent lock file. The lock file exists on disk, making it visible across processes regardless of whether those processes share memory space. Any process that needs to modify the session index must acquire the lock file first.

This replaces an in-process coordination mechanism that was invisible to external processes. The persistent lock file approach is less efficient than an in-memory lock in single-process scenarios, but it is correct in multi-process ones.

The practical effect: developers running two Codex windows simultaneously should see fewer cases where session history from one window disappears or becomes corrupted in the other.

---

## Workspace Visibility in /status

PR #57366, also from Eric Traut, adds the selected ChatGPT workspace name and account ID to the output of the `/status` command.

This is a small change with outsized utility in enterprise deployments where developers belong to multiple organisations or operate across personal and work accounts. Previously, confirming which workspace Codex was operating under required exiting to account settings. The workspace is now visible inline in `/status`, alongside the existing model and permission information.

---

## Rust Toolchain: Pinned to 1.95.0

PR #57373 (Anthony Tafoya) pins the Rust toolchain to 1.95.0. This is a build-infrastructure change rather than a user-facing feature. Toolchain pinning reduces build variability across different developer environments and CI configurations, and simplifies reproducibility investigations when builds fail on specific platforms.

---

## The v0.165.0-alpha.3 Window

Before the 0.166 branch opened, v0.165.0-alpha.3 landed at 21:54 UTC on 26 October. Three PRs are worth noting:

**PR #57325 — Granular approval templates for enterprise policies**

Enterprises can now define fine-grained approval rules per action category within the policy configuration. Previously, approval templates operated at the session level. Category-level granularity allows, for example, file-write approvals to follow a different policy from network-request approvals within the same session.

**PR #57313 — OAuth delivery defaults to Bearer tokens**

Bearer token delivery is now the default for OAuth-authenticated MCP connections. The previous mixed-delivery behaviour required explicit configuration to produce consistent token formats across different MCP server implementations. This change eliminates a configuration class that existed primarily to accommodate legacy behaviour.

**PR #57307 — Security fence library CVE patch (v0.12.2)**

A dependency security patch. No user-visible behaviour change; applies the current upstream security fix to the sandboxing layer.

---

## What the 0.166.x Sprint Tells Us

Three sprint themes are visible from alpha.1:

**Safety UX is the next major user-facing investment.** The recourse work from Eric Traut is not a minor polish item. Presenting users with actionable alternatives after a policy block is a substantive rethinking of the safety interaction model. Expect further PRs in this area as the sprint progresses — recourse personalisation, recourse acceptance feedback, and potentially recourse logging for compliance audit trails.

**Model catalogue management is becoming continuous.** Terra arrives in alpha.1, one sprint after Sol. The cadence suggests OpenAI is now maintaining a rolling model upgrade path within Codex rather than waiting for major version milestones. For teams maintaining model selection configuration, this means more frequent evaluation cycles.

**Multi-process reliability is being taken seriously.** The session lock PRs are infrastructure, not features. They fix a real correctness problem that affects developer workflows built around parallel Codex instances. The investment signals that OpenAI is treating multi-window, multi-process usage as a supported configuration rather than an edge case.

---

## Key Issues Still Open

The arrival of v0.166.0-alpha.1 does not resolve these carry-forward signals:

| Signal | Days open | Status |
|--------|-----------|--------|
| GPT-6.1 Sol AGENTS.md compliance regression | 19 days | Open, no official response |
| Projects data-loss after updates | Active | SQLite records exist; projects disappear from UI |
| VS Code workspace scoping (#25319) | Active | 125 upvotes, no fix landed |
| Windows file deletion P0 | 4 days | Open |
| Linux OOM 47-49 GB RAM spike | 4 days | Open |

---

## Using v0.166.0-alpha.1

Alpha builds are not recommended for production use. To run v0.166.0-alpha.1:

```bash
# Check your current version
codex --version

# Update to the latest release (includes pre-release)
npm install -g @openai/codex@alpha

# Confirm the version
codex --version  # Should show 0.166.0-alpha.1 or later
```

To test Terra model availability once your alpha build is running:

```bash
codex /model
# Look for gpt-6.1-terra in the model list
```

Safety recourse prompts appear automatically when a policy block fires in an interactive TUI session. No configuration is required.

---

## Summary

v0.166.0-alpha.1 opens a new sprint thread while v0.165.x observability work continues in parallel. The three core changes — GPT-6.1-Terra model support, safety policy recourse in the TUI, and multi-process session reliability — address different layers of the user experience. Terra expands the model options for teams affected by the Sol compliance regression. Safety recourse makes blocked sessions recoverable rather than terminal. Session index serialisation makes multi-window workflows reliable rather than accident-prone.

Sprint day 23 of 28. The 0.166.x arc is just beginning.

---

*Written with AI assistance. This newsletter is researched and drafted by Andy, an AI assistant. Daniel Vaughan reviews and directs all published content.*
