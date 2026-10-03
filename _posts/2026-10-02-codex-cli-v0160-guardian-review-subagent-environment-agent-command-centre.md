---
title: "Codex CLI v0.160.0: Guardian Review Gets Conversation History, Subagents Inherit Environments"
parent: "Articles"
nav_order: 1169
date: 2026-10-02T07:00:00+00:00
last_modified_at: 2026-10-03T18:11:47+01:00
tags: ["codex-cli", "v0.160.0", "guardian", "multi-agent", "subagents", "security", "agent-command-centre", "sqlite", "linux", "windows"]
---

# Codex CLI v0.160.0: Guardian Review Gets Conversation History, Subagents Inherit Environments


---

Codex CLI v0.160.0 landed on 1 October 2026 as the first stable release in the 0.160 line.[^1] The release did not ship through a conventional alpha-to-stable promotion — the 0.160 alpha series ran in parallel and was superseded by the stable tag directly. The result is a release that bundles a significant set of Guardian review improvements, subagent environment handling, Linux UX fixes, SQLite performance work, and Windows sandbox hardening.

The headlining change is Guardian: the code-review and policy-enforcement layer that assesses Codex agent actions during and after execution. v0.160.0 gives Guardian reviewers access to conversation history for the first time, preserves encrypted messages through the review pipeline, and introduces handoff-aware context selection for multi-agent sessions.

## Guardian Review: Conversation History

Guardian review in previous releases operated on a snapshot of the agent's current turn. Reviewers — whether human or automated policy — could see the immediate action being evaluated but had no reliable way to retrieve what the agent had said or decided earlier in the session.

v0.160.0 adds an optional tool that allows Guardian reviewers to search and read earlier messages in the conversation.[^2] The tool is gated behind a permission check; sessions must explicitly enable it. When enabled, a reviewer can query message history by time range, author, or content, and include relevant earlier context in its assessment.

This changes the power of Guardian considerably. Policy rules can now detect patterns that span multiple turns — for example, an agent that requests escalating permissions across a long session, or one that contradicts an earlier instruction it claimed to have followed.

## Guardian Review: Encrypted Message Preservation

Multi-agent Codex sessions involve agent-to-agent messages that may be encrypted in transit. Prior to v0.160.0, these encrypted messages were not reliably preserved when a Guardian review was triggered: the review pipeline could drop, reorder, or truncate the encrypted content.

v0.160.0 fixes this by threading encrypted `agent_message` items through both synchronous reviews and asynchronous scoring, preserving their order, author, recipient, and ciphertext intact.[^2] The encrypted messages also appear correctly in transcript budgets and message ordering, so they do not distort token counts or confuse the review model about sequence.

This matters for enterprise deployments where inter-agent communication is encrypted for compliance reasons. Guardian reviews on those sessions were previously unreliable; they should now be complete.

## Guardian Review: Handoff-Aware Context

When Codex runs a multi-agent session — an orchestrator spawning worker subagents via `spawn_agent`, `send_message`, or `followup_task` — the Guardian review previously received a flat view of all messages across all agents. Distinguishing which messages belonged to which subagent required the reviewer to reconstruct the handoff graph manually.

v0.160.0 introduces `guardian_root_handoff_context`, a disabled-by-default feature that selects worker-specific root evidence for each Guardian review.[^2] When enabled, the system uses the recorded handoff calls to identify which root messages preceded and followed each handoff, then provides the reviewer with a focused context slice that reflects the worker's actual scope rather than the full session transcript.

The practical effect: Guardian reviews in multi-agent sessions become more accurate and less noisy. A policy check on a specific worker agent now sees the messages relevant to that agent rather than the entire orchestration tree.

## Agent Command Centre: Pagination for Older Tasks

The agent command centre — the TUI panel listing active and recent tasks — previously showed only the most recent tasks and had no navigation for older entries. In sessions with long-running agent queues or high task throughput, tasks would scroll off the visible list with no way to retrieve them without exiting the TUI.

v0.160.0 adds a keyboard-accessible "Show more" action to the command centre.[^1] Activating it loads an additional page of older tasks into the list. The pagination is incremental: each "Show more" loads the next batch rather than loading all tasks at once, which avoids performance issues in sessions with hundreds of historical entries.

## Subagent Environment Inheritance

When Codex spawns a subagent, the parent's pending environment configuration — environment variables staged for the next task, sandbox overrides, workspace defaults — was previously not transferred. The subagent started from a clean environment, which meant configurations set at the orchestrator level did not propagate down to workers automatically.

v0.160.0 changes this: pending environments now transfer to spawned subagents with configuration inheritance.[^2] Orchestration patterns that rely on shared environment context across the agent tree — API keys, working directory overrides, model selections — now propagate without requiring each worker to be configured independently.

This removes a class of orchestration bugs where worker agents silently used default configuration instead of the orchestrator's intended settings.

## Linux: Middle-Click Paste for X11 Terminals

Codex CLI's fullscreen TUI previously ignored the X11 primary selection — the buffer populated by highlighting text in any X11 application. Middle-clicking in the Codex TUI on Linux did nothing.

v0.160.0 adds middle-click paste support for Linux X11 terminals.[^1] Highlighted text from any X11 application can now be pasted into the Codex prompt area with a middle click. This aligns Codex's input behaviour with standard X11 conventions and is particularly useful for workflows that involve copying code fragments from adjacent terminal windows.

## Session Start Outside Projects

Previously, launching Codex CLI outside a recognised project directory — one without an `AGENTS.md` or a workspace root marker — prompted Codex to ask for a project context before proceeding. This friction blocked quick one-off queries and scripted invocations where project context was irrelevant.

v0.160.0 allows sessions to start outside projects using workspace defaults.[^1] When no project context is detected, Codex falls back to configured workspace defaults rather than halting. The defaults cover model selection, sandbox policy, and tool permissions — the same configuration used by project sessions, but applied without requiring a project root.

## Plan Mode Hint in Fullscreen Status Line

The fullscreen TUI status line now displays a contextual hint when plan mode is active: `Plan mode (shift+tab to cycle)`.[^2] This addresses a common point of confusion for new users who enter plan mode and cannot immediately see how to exit or cycle modes. The hint is displayed inline in the status bar without requiring a separate help panel.

## Plan Tier Label Standardisation

OpenAI has standardised the naming of ChatGPT subscription tiers used in the Codex TUI. The previous labels were non-intuitive internal identifiers:

| Old Label | New Label |
|-----------|-----------|
| Pro Extra | Pro 200 |
| Pro Standard | Pro 100 |
| Pro Max | Pro 500 |

The numbers correspond to the monthly usage allowance at each tier. The change affects the TUI account panel and any configuration references to subscription tier names.[^2]

## SQLite Performance: Background Vacuum and Connection Optimisation

Codex CLI maintains a local SQLite database for session logs and task history. Two performance issues have been addressed in v0.160.0:

**Background vacuum**: the logs database now runs incremental page reclamation (vacuum) in the background with configurable thresholds.[^2] Previously the database could accumulate deleted pages indefinitely, growing the file size without recovering space. The background vacuum runs opportunistically between sessions.

**Connection setup optimisation**: connection setup stalls and logging overhead that blocked write transactions have been eliminated.[^2] In high-throughput agent sessions with rapid task creation and logging, this could cause visible latency in the TUI. The fix applies to both the primary session database and the logs database.

## Windows Sandbox Hardening

Three Windows-specific improvements ship in v0.160.0:

**PowerShell fallback**: extended fallback discovery to Microsoft eXecution Containers (MXC) sandbox environments.[^2] Previously, Codex could fail to locate PowerShell when running inside MXC sandboxes, breaking Windows-based agent tasks that relied on PowerShell execution.

**Long-path ACL repairs**: enhanced support for extended-length paths using handle-based security updates.[^2] Windows paths exceeding 260 characters — common in deeply nested project structures — could fail ACL permission repairs. The fix uses handle-based security operations rather than path-based ones, which are not subject to the length limit.

**Typed error codes**: replaced free-form error text in security-sensitive paths with typed error codes.[^2] This prevents credential or configuration values from appearing in error messages, reducing the risk of credential exposure through log output or error reporting.

## Diff Display Optimisation for Guardian Sessions

Codex CLI's diff display previously attempted to discover the remote Git configuration for every session before rendering diffs. In Guardian review sessions — which may have no Git remote configured — this caused stalls while the discovery timed out.

v0.160.0 skips remote Git discovery when operating in a Guardian session.[^2] Diffs render immediately without waiting for a remote timeout. This is a targeted fix but meaningfully improves the responsiveness of Guardian review sessions in environments with restricted or absent Git remote access.

## Summary

| Area | Change |
|------|--------|
| Guardian | Conversation history tool for reviewers (opt-in) |
| Guardian | Encrypted agent message preservation through review pipeline |
| Guardian | Handoff-aware context selection for multi-agent reviews (disabled by default) |
| Agent command centre | Pagination for older tasks |
| Subagents | Pending environment inheritance from parent |
| Linux | Middle-click paste for X11 terminals |
| Sessions | Start outside projects using workspace defaults |
| TUI | Plan mode hint in fullscreen status line |
| TUI | Plan tier label standardisation (Pro 100/200/500) |
| SQLite | Background vacuum + connection optimisation |
| Windows | PowerShell fallback, long-path ACL, typed error codes |
| Diffs | Skip remote Git discovery in Guardian sessions |

v0.160.0 is available via `npm install -g @openai/codex` or `brew upgrade codex`.

---

[^1]: OpenAI, "ChatGPT & Codex changelog", 1 October 2026. <https://learn.chatgpt.com/docs/changelog>
[^2]: JJLiebig, "Integrate upstream rust-v0.160.0", GitHub Pull Request #219, 1 October 2026. <https://github.com/JJLiebig/codex/pull/219>
