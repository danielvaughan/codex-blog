---
title: "OpenAI DevDay 2026: Codex Cloud, the /agents View, GPT-6.1 Sol, Code Review, and Security Cloud"
parent: "Articles"
nav_order: 1173
date: 2026-10-08T07:00:00+00:00
last_modified_at: 2026-10-09T10:26:21+01:00
tags: ["codex-cli", "devday", "openai", "codex-cloud", "agents-view", "gpt-6", "sol", "code-review", "security-cloud", "multi-agent"]
---

# OpenAI DevDay 2026: Codex Cloud, the /agents View, GPT-6.1 Sol, Code Review, and Security Cloud


---

OpenAI held DevDay 2026 on 29 September in San Francisco.[^1] Five announcements defined the event for Codex CLI developers: Codex Cloud (remote task execution with the laptop closed), the /agents view (parallel agent management from a single CLI session), GPT-6.1 Sol as the new default model, Code Review (automated PR review for GitHub and GitLab), and Codex Security Cloud (scheduled full-repository vulnerability scanning). Taken together, they represent a shift in how OpenAI is positioning Codex — from a local coding assistant into a distributed agent platform that runs whether or not a developer is at their desk.

This article covers each announcement in turn, explains what changed in the CLI, and provides the practical configuration details you need to use each feature.

## Codex Cloud: Running Tasks While Your Laptop Is Closed

The most architecturally significant announcement at DevDay 2026 was Codex Cloud: the ability to run Codex sessions remotely, on OpenAI's infrastructure, with the laptop powered down.[^2] Prior to this announcement, all Codex CLI sessions ran in a local sandbox on the developer's machine. Closing the laptop killed the session.

Codex Cloud changes this by moving the execution environment to a managed remote container. You start a task from the CLI — or from ChatGPT — and it runs to completion in the cloud regardless of local machine state. Cross-device handoff is real: a task started on a desktop can be monitored or continued from a phone. Named sessions replace the worktree-per-task model for managing continuity.

Three differences from local sessions matter for AGENTS.md design.

**Filesystem access is narrower.** In a cloud session, Codex has access to the repository it was given at session start and nothing else. Local paths, home directory configurations, and adjacent project directories are not available. AGENTS.md directives that reference local filesystem conventions — `~/.config/`, relative paths to tools installed outside the repo — will silently fail in cloud mode. All paths referenced in AGENTS.md should be repository-relative or use the `CODEX_WORKSPACE` environment variable.[^3]

**Secret management works differently.** Local sessions typically inherit `.env` files and shell environment variables. Cloud sessions have no access to the developer's shell. Secrets must be declared in the project's cloud environment configuration, accessible via the `/cloud env` command, and injected at task start. The practical implication: treat every Codex Cloud session as you would a CI job, not as an extension of your local shell.[^3]

**Session continuity uses named sessions.** Where local workflows use git worktrees to isolate parallel tasks, cloud sessions use named session identifiers. You can create, attach to, and hand off named sessions across devices. The `--session` flag on the CLI maps to the same identifier in ChatGPT's agent view.[^2]

For teams already running Codex in CI via `codex run --headless`, Codex Cloud is a natural extension: the same headless execution model, but with OpenAI managing the container lifecycle rather than your CI provider.

## The /agents View: Parallel Agent Management from One Pane

The second major announcement was the /agents view: a consolidated TUI panel for managing multiple parallel Codex sessions from a single CLI window.[^4] Prior to DevDay 2026, running five parallel agent sessions required five separate terminal windows or tmux panes with no shared state between them.

The /agents view changes this. Entering `/agents` at the CLI prompt opens a panel showing all active sessions: their names, current status (running, waiting for input, complete, interrupted), and progress indicators. From the panel you can interrupt a specific agent, attach to its conversation history, or spin up a new session — all without leaving the primary TUI window.

Key controls:
- **`/fork`** (added in v0.157) — create a new parallel session branched from the current conversation context.[^5]
- **`instant_interrupt`** (added in v0.159) — pause a running agent immediately without waiting for the current tool call to complete.[^6]
- **Status indicators** — each entry in the /agents panel shows a colour-coded state: green (running), amber (waiting), grey (complete), red (error or interrupted).

A practical workflow: run five agents in parallel — three implementing features in separate worktrees, one running tests, one updating documentation — and monitor all five from the /agents panel. When the test runner signals a failure, use `instant_interrupt` to pause the implementation agents, attach to the test runner to review the failure, then resume.

The /agents view requires no configuration changes. It is available in all Codex CLI sessions from v0.159.0 onwards.

## GPT-6.1 Sol: Near-Astra Performance at One-Fifth the Price

DevDay 2026 made GPT-6.1 Sol the new default model for Codex CLI, shipping in v0.159.1.[^7] The pricing signal is significant: GPT-6.1 Sol is priced at \$2 per million input tokens and \$10 per million output tokens — representing a 90 per cent reduction in cached input costs relative to Astra's standard tier (\$0.10/M vs \$1.00/M cached input). Bijan Bowen, writing in the OpenAI engineering blog in September 2026, noted that running a full test suite with GPT-6.1 Sol consumed only 3 per cent of a weekly token limit.[^8] That finding changes cost assumptions for any team currently defaulting to Astra for standard agentic work.

The routing logic is straightforward. GPT-6.1 Sol handles the majority of standard coding tasks — feature implementation, refactoring, test generation, documentation updates — at high quality and very low cost. GPT-6 Astra (Light, Medium, or Extra High) remains the tier for work that requires cross-context memory, multi-step reasoning over long sessions, or tasks where the quality ceiling of Sol is insufficient. GPT-6 Luna is the appropriate choice for lightweight subagent roles, simple edits, and rate-limit fallback.

| Tier | Model | Relative cost | When to use |
|------|-------|---------------|-------------|
| Astra | GPT-6 Astra | Highest | Long sessions, cross-context memory, complex reasoning |
| Sol | GPT-6.1 Sol | Mid (90 per cent cheaper cached input vs Astra standard) | Standard coding tasks, daily driver |
| Luna | GPT-6 Luna | Lowest | Subagent roles, lightweight edits, rate-limit fallback |

To pin Sol explicitly in `config.toml`:

```toml
model = "gpt-6.1-sol"
```

For sessions that need Astra's memory features on a per-task basis, the `--model` flag overrides the config file:

```bash
codex --model gpt-6-astra-medium "Audit this module for security issues"
```

The version that shipped GPT-6.1 Sol as the default was v0.159.1, released in the days following DevDay. Earlier v0.15x releases used GPT-6 Sol (without the `.1` revision). If you pin a specific model identifier, verify you are using `gpt-6.1-sol` rather than `gpt-6-sol` for the DevDay default.

## Code Review: Automated PR Review for GitHub and GitLab

Code Review was the fourth DevDay 2026 announcement: Codex can now be connected to GitHub and GitLab repositories and will automatically review pull requests when triggered.[^9] The feature integrates with the pull request review workflow natively — Codex posts its review as a code review comment with inline annotations, not as a bot comment appended to the PR description.

The review is triggered by adding Codex as a reviewer. In GitHub, that means requesting a review from the Codex integration account; in GitLab, the equivalent workflow applies. Codex reads the diff, the PR description, and the AGENTS.md file at the repository root (if present) and applies any review policies defined there alongside its standard review logic.

Review capabilities at launch include:
- **Logic correctness** — identifying incorrect handling of edge cases, off-by-one errors, null-safety gaps
- **Security patterns** — flagging common vulnerability patterns (injection, improper input validation, credential handling)
- **Style and maintainability** — identifying deviations from patterns established in the existing codebase
- **Test coverage** — noting when changed code paths lack corresponding test coverage

One practical constraint: Code Review at launch does not execute code. Codex reads the diff and the repository context but does not run the test suite, build the project, or verify runtime behaviour. It is a static review, not a dynamic one. Treating it as a first-pass filter — catching obvious issues before a human reviewer invests time — is the appropriate mental model.

AGENTS.md can be used to guide the review focus. Adding a `## Code Review` section with explicit priorities (for example, "always check for SQL injection in any file under `src/db/`") directs Codex's attention to the areas your team cares most about.

## Codex Security Cloud: Scheduled Repository Scanning

The fifth announcement was Codex Security Cloud: scheduled, full-repository vulnerability scanning that runs continuously in the background with the laptop closed.[^10] Where Code Review operates on a single pull request when triggered, Codex Security Cloud scans the entire codebase on a schedule — daily, weekly, or continuous — and prepares fix candidates for detected issues.

The output of a Security Cloud scan is not just a report. For each identified vulnerability, Codex prepares a fix: a proposed change to the repository that resolves the issue. The fix is presented as a pull request (or a patch, depending on configuration) for developer review. This positions Codex not merely as a detection tool but as a remediation pipeline.

The workflow is:

1. Codex Security Cloud runs a scheduled scan against the configured repository.
2. Issues are classified by severity (Critical, High, Medium, Low) and mapped to CVE or CWE identifiers where applicable.
3. For each issue, a fix candidate is prepared in a Codex Cloud session and surfaced as a proposed PR.
4. Developers review and merge (or reject) the fix through the standard PR workflow.

This model differs meaningfully from GitHub Advanced Security and Dependabot. Advanced Security surfaces alerts and expects humans to write the fix. Dependabot prepares fixes for dependency upgrades but does not address code-level vulnerabilities. Codex Security Cloud prepares fixes for code-level issues — the class of vulnerability that neither of the existing tools addresses automatically.[^10]

At launch, Codex Security Cloud is available on Pro, Business, Enterprise, and Edu plans. Configuration is managed through the Codex dashboard under Security > Scheduled Scans. Repository connection reuses the same OAuth integration used by Code Review.

## What the Five Announcements Add Up To

Read individually, each DevDay 2026 announcement is a meaningful feature addition. Read together, they describe a consistent architectural direction: Codex is becoming infrastructure, not just a tool.

Codex Cloud means execution is no longer tethered to a developer's local machine. The /agents view means managing multiple concurrent agents becomes a first-class operation rather than a terminal-juggling exercise. GPT-6.1 Sol's pricing makes it economically viable to run agents continuously rather than selectively. Code Review makes Codex a participant in every PR, not just a tool invoked on demand. Security Cloud makes Codex a continuous security presence in the repository, not a one-time audit.

For a team currently using Codex primarily for greenfield feature work, the DevDay announcements add three new operational modes: background security monitoring, continuous code review, and cloud-resident long-running agents. None of these require significant changes to existing Codex workflows. They extend the surface area of what Codex does without changing how it is configured for existing use cases.

The practical upgrade path: update to v0.159.1 or later to get GPT-6.1 Sol as the default. Enable the /agents view with `/agents` in any active session — no configuration required. For Code Review and Security Cloud, connect the GitHub or GitLab integration through the Codex dashboard. For Codex Cloud, review your AGENTS.md files for local path dependencies before migrating any long-running tasks to cloud execution.

---

[^1]: OpenAI, "DevDay 2026," San Francisco, 29 September 2026. <https://openai.com/devday>
[^2]: Codex Cloud announcement, DevDay 2026 keynote, 29 September 2026.
[^3]: "Codex in the Cloud: AGENTS.md Patterns That Travel," Codex CLI knowledge base, October 2026 (forthcoming).
[^4]: /agents view announcement, DevDay 2026 keynote, 29 September 2026.
[^5]: Codex CLI v0.157.0 release notes — `/fork` shortcut for parallel session creation. September 2026.
[^6]: Codex CLI v0.159.0 release notes — `instant_interrupt` flag for immediate agent pause. September 2026.
[^7]: Codex CLI v0.159.1 release notes — GPT-6.1 Sol set as default model. September 2026.
[^8]: Bowen, B., "GPT-6.1 Sol in Production: Token Budget Data from a Real Test Suite," OpenAI Engineering Blog, September 2026.
[^9]: Code Review announcement, DevDay 2026 keynote, 29 September 2026.
[^10]: Codex Security Cloud announcement, DevDay 2026 keynote, 29 September 2026.
