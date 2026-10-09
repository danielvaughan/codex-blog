---
title: "Voice on by Default: How v0.156.0 Changes the Codex CLI's Identity"
parent: "Articles"
nav_order: 1168
date: 2026-09-22T08:00:00+00:00
last_modified_at: 2026-10-09T19:06:14+01:00
tags: ["codex-cli", "voice", "v0.156.0", "tui", "fullscreen", "usage-dashboard", "worktrees", "mermaid", "multimodal", "identity-shift"]
---

# Voice on by Default: How v0.156.0 Changes the Codex CLI's Identity


---

Codex CLI v0.156.0, released 22 September 2026, is not a typical feature release.[^1] Several of its changes are individually straightforward — a new keyboard shortcut here, a dashboard command there. Together they reframe what Codex CLI is. Before v0.156.0, Codex was a text-based coding assistant with optional voice and worktrees you had to configure manually. After v0.156.0, it is a multimodal agentic interface with voice on, parallel isolation on, and native visual output — all by default. The terminal is still there; it is no longer the defining characteristic.

This article covers each of the major v0.156.0 changes, explains the ergonomic gap each one closes, and gives the practical configuration decisions that matter to teams already running Codex in production.

## Voice Conversations Are Now the Default

The most consequential single change in v0.156.0 is the promotion of voice from experimental opt-in to the default interaction mode. Before this release, using voice required a flag or an explicit enable in `config.yaml`. Now the voice runtime starts with every session. The F8 key toggles listening state; `/voice settings` opens the language and microphone picker; audio runtimes for Linux and Windows ship bundled, removing the runtime installation step that blocked adoption on non-macOS platforms.[^1]

What this means in practice: you no longer think of voice as a separate mode you switch into. It is available at F8 whenever you want it, in the same session and the same context as your typed conversation. Transcript review happens before execution — Codex does not act on voice input until you confirm the transcript, which preserves the deliberate execution model that teams rely on for production-adjacent sessions.

The ergonomic shift is clearest in long-horizon tasks. When you are reading code across multiple files, narrating observations is faster than typing them. When you are waiting for a test run, you can ask a follow-up question verbally without breaking flow. The feature has existed for months; making it default removes the psychological barrier of "I should set that up sometime."

**Configuration note:** If you are running Codex in a shared terminal environment or CI context where audio capture would fail silently, set `voice.enabled = false` in `config.yaml`. The option exists precisely for headless use cases. The default is optimised for developer workstations; it is not forced on every invocation.

## Fullscreen TUI: `/tui`

The `/tui` command opens the fullscreen transcript viewer, introduced as a stable feature in v0.156.0.[^1] The interface adds transcript search (scroll through a long session and find an earlier decision), mouse selection, and right-click copying — features the standard terminal layout cannot offer because the terminal itself owns those interactions.

The practical use case is long sessions. In a standard terminal, output from fifty tool calls ago is gone — you can scroll back but you cannot search. The fullscreen TUI treats the session as a searchable log. This matters most for overnight runs reviewed in the morning, and for debugging sessions where you need to find the exact point at which an assumption was introduced.

Six new terminal themes ship alongside the fullscreen mode, covering both light and dark preferences and high-contrast accessibility variants. Theme selection is in `/voice settings` alongside language preferences — the settings picker now consolidates display and audio preferences rather than spreading them across `config.yaml`.

## Mermaid Diagrams and Display Equations

Codex CLI v0.156.0 renders Mermaid diagrams and display equations directly in the TUI response area.[^1] Before this release, if you asked Codex to diagram an architecture or write a derivation, the output was a fenced code block — readable but not rendered.

Rendered Mermaid output is practically useful for architecture questions. Ask Codex to diagram the relationship between your services and the output is a flowchart, not a text description of one. The `/tui` fullscreen mode is the natural companion: the diagram renders at full width rather than being constrained to a narrow terminal column.

Rendering toggles for Mermaid, equations, and tables are independent — you can enable Mermaid while leaving table rendering off if your terminal font handles the box characters poorly. The toggles live in session config and persist across sessions.

## Worktrees On by Default

Worktree support was introduced as an opt-in API in v0.154.0.[^4] v0.156.0 makes it the default. Every new session can now create a native git worktree directly from the agents command centre — no `git worktree add` scripting, no environment variable management, no manual rollback setup.

The architectural implication matters more than the UX convenience. Worktrees on by default signals that parallel-agent workflows are now the expected operating mode, not an advanced option. The agents command centre lists running sessions, their worktree status, and their task filter (by status: running, waiting, complete). You can create a new isolated session, assign it to a branch, and have it running in parallel with your current session in four keypresses.

For teams already using manual worktree scripting: the managed path and the manual path coexist. The `WorktreeManager` API that underpins the default does not prevent direct `git worktree` calls. Teams with custom setup scripts can migrate incrementally or continue using them unchanged. The default improves the out-of-the-box experience without breaking established workflows.

## The `/usage` Dashboard

`/usage` opens a real-time spend dashboard scoped to the current account: token totals, plugin and skill activity, and usage trend lines.[^1] Before v0.156.0, spend visibility required the OpenAI web console or third-party logging. The `/usage` dashboard closes that gap at the TUI layer.

The practical habit is to run `/usage` at the start and end of each sprint. The delta tells you what a single focused coding session costs at the model and plugin level — which is the input you need to tune rollout token budgets and multi-agent concurrency. The dashboard is account-level, not session-level, so it does not give per-task breakdown; for session-level cost attribution, `codex queue` status output remains the right primitive. The two work together rather than replacing each other.

The `/usage` dashboard deserves more detailed treatment than a paragraph — it has its own article in the backlog. The key point here is that spend visibility is now a first-class TUI feature, not a workaround.

## What the Changes Add Up To

Individual feature summaries miss the pattern. Read together:

- Voice is default. Typing is optional.
- The output layer renders diagrams and equations. Plain text is a fallback, not the target.
- Parallel isolation is default. Single-session operation is still available but is no longer the implicit baseline.
- Spend is visible without leaving the tool.

This is not a list of additions to a coding assistant. It is a different category of tool: a multimodal agentic interface that runs locally, operates across parallel sessions by default, speaks and listens, visualises output natively, and exposes its own cost at the TUI layer. The terminal origin is still visible in the interaction model — transcript review before execution, explicit file scope, `AGENTS.md` configuration — but the surface has expanded well beyond it.

Teams evaluating Codex CLI for new use cases should benchmark against the v0.156.0 defaults, not earlier versions. Teams already using Codex should audit their `config.yaml` entries for voice and worktree settings: most existing overrides that disabled experimental features should be removed, because the features are no longer experimental.

## Summary

v0.156.0 promotes voice, fullscreen TUI, and worktree isolation from opt-in to default. It adds Mermaid and equation rendering, six new themes, task filtering by status, and the `/usage` spend dashboard. None of these changes is individually a breakthrough. Together they mark the point at which Codex CLI's identity as a multimodal agentic interface became the stable baseline, not a preview.

---

[^1]: OpenAI, "Codex CLI v0.156.0 release notes," 22 September 2026. Voice on by default (F8 toggle, `/voice settings`, bundled audio runtimes); optional fullscreen UI (`/tui`); `/usage` analytics dashboard; worktree support enabled by default; task filtering; six new terminal themes; Mermaid diagrams and display equations. https://developers.openai.com/codex/changelog
[^2]: OpenAI, "Codex CLI v0.156.1 release notes," 23 September 2026. GPT-6 Sol and Luna added to model picker; rate-limit default updated to Luna. https://developers.openai.com/codex/changelog
[^4]: OpenAI, "Codex CLI v0.154.0 release notes," September 2026. `WorktreeManager` API introduced; native git worktree creation and teardown managed by Codex. https://developers.openai.com/codex/changelog
[^5]: OpenAI, "Codex CLI v0.150.0 release notes," 26 August 2026. `@task-name` inter-session messaging primitive introduced for cross-task coordination. https://developers.openai.com/codex/changelog
