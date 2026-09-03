---
title: "Claude Code Routines: Autonomous Scheduled Agent Runs on Anthropic Cloud"
description: "Anthropic launched Claude Code Routines on 14 April 2026 — saved Claude Code configurations (a prompt, one or more repositories."
type: Technical Article
timestamp: 2026-04-14T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-04-14-claude-code-routines-launch"
tags: ["claude-code", "routines", "competitor-update", "autonomous-agents", "scheduled-tasks", "cloud-execution", "github-triggers", "mcp-connectors"]
date: 2026-04-14T09:00:00+00:00
last_modified_at: 2026-09-03T10:08:39+01:00
---
![Sketchnote diagram for: Claude Code Routines: Autonomous Scheduled Agent Runs on Anthropic Cloud](/sketchnotes/articles/2026-04-14-claude-code-routines-launch.png)

# Claude Code Routines: Autonomous Scheduled Agent Runs on Anthropic Cloud

**Source:** [Anthropic Documentation](https://docs.anthropic.com/en/docs/claude-code/routines) · [SiliconAngle Coverage](https://siliconangle.com/2026/04/14/anthropics-claude-code-gets-automated-routines-desktop-makeover/)
**Author:** Anthropic
**Date saved:** 2026-04-16
**Content age:** Current — announced April 14, 2026

---

## Summary

Anthropic launched Claude Code Routines on 14 April 2026 — saved Claude Code configurations (a prompt, one or more repositories, and a set of MCP connectors) that run autonomously on Anthropic-managed cloud infrastructure. Routines can be triggered by a recurring schedule, an HTTP API call, or a GitHub event (pull request or release), and they keep running even when the developer's laptop is closed. The feature is in research preview; daily run limits apply by tier: Pro 5/day, Max 15/day, Team/Enterprise 25/day.

---

## Key Points

- **What a routine is:** a saved Claude Code configuration — prompt + repos + connectors — packaged once and run automatically on Anthropic's cloud
- **Runs when your laptop is off:** routines execute on Anthropic-managed cloud infrastructure, not the developer's machine
- **Three trigger types:** scheduled (hourly/daily/weekly/custom cron), API (HTTP POST to a per-routine endpoint with bearer token), and GitHub events (pull_request or release, with rich filtering)
- **Triggers can combine:** a single routine can fire on a schedule, via API, and on GitHub events simultaneously
- **Tier limits:** Pro 5/day, Max 15/day, Team/Enterprise 25/day; overage available with extra-usage billing enabled
- **Research preview status:** behaviour, limits, and the API surface may change; the `/fire` endpoint requires the `experimental-cc-routine-2026-04-01` beta header
- **Creation surfaces:** web UI at claude.ai/code/routines, CLI via `/schedule`, or Desktop app via New Remote Task
- **Full autonomy:** no permission-mode picker, no approval prompts during a run — the session can run shell commands, use skills from the cloned repo, and call any included connectors
- **Branch safety:** Claude can only push to `claude/`-prefixed branches by default; unrestricted branch pushes can be enabled per-repository
- **GitHub trigger filters:** author, title, body, base/head branch, labels, draft status, merged status, fork status — with operators including regex matching
- **MCP connectors included:** routines can read/write to Slack, Linear, Google Drive, and other connected services during each run
- **Environments:** configurable network access, environment variables, and setup scripts per routine
- **Example use cases from Anthropic:** backlog maintenance, alert triage, bespoke code review, deploy verification, docs drift detection, library porting across SDKs
- **API response:** returns a session ID and URL; sessions can be watched live, reviewed, or continued manually
- **Desktop redesign announced alongside:** integrated terminal, faster diff viewer, in-app file editor, expanded preview area, multiple session support

---

## Competitive Implications for Codex CLI

This is the most significant competitive move from Anthropic since Claude Code went open-source. Routines establish a new category — **cloud-hosted autonomous agent execution** — that Codex CLI has not entered yet:

1. **Cloud execution gap.** Codex CLI runs locally or via `codex exec` in CI pipelines, but has no equivalent of a managed cloud runtime that persists across laptop closures. The closest analogues are Codex Cloud tasks (which require an active session) and community tools like NanoClaw/OpenClaw for always-on orchestration.

2. **Trigger sophistication.** Codex CLI's trigger model is limited to GitHub Actions integration and manual invocation. Routines offer first-party schedule, API, and GitHub event triggers with granular PR filtering — a superset of what Codex users achieve today through custom GitHub Actions YAML.

3. **Goal Mode direction.** OpenAI's Goal Mode PRs in the codex-rs repository suggest they are heading toward similar autonomous execution, but nothing shipped yet. Routines give Anthropic a first-mover advantage in the "set it and forget it" agent workflow space.

4. **Connector ecosystem.** By including MCP connectors (Slack, Linear, Google Drive, etc.) in routine runs, Anthropic makes routines a full workflow automation platform — not just a code agent. This is closer to Zapier-meets-AI-coding than anything Codex CLI offers today.

5. **Enterprise appeal.** The 25 runs/day Team/Enterprise tier, combined with branch safety defaults and environment isolation, makes routines immediately deployable in regulated environments. Codex CLI's enterprise story is currently centred on the Guardian AI sandbox and TOML-based approval policies — effective but requiring more setup.

6. **Pricing pressure.** Routine usage counts against existing subscription limits, meaning Pro users get 5 autonomous runs per day included. This is a strong value proposition that could pull developers toward Claude Code for autonomous workflows while using Codex CLI for interactive coding.

---

## Book/Article Impact

- **ch05 (Competing Tools)** needs updating — Routines represent a new capability category that must be covered in the Claude Code section; the current competitive analysis predates cloud-hosted autonomous execution
- **ch16 (Hooks)** — worth contrasting Codex CLI's hook-based automation (local, synchronous, developer-present) with Routines (cloud, asynchronous, unattended)
- **Premium article 06 (Three Terminals)** should note that Routines add a fourth execution surface (cloud-autonomous) beyond the three terminal pattern
- **Premium article 12 (Scaling Playbook)** — Routines are directly relevant to enterprise adoption; the "always-on agent" pattern they enable is a key scaling differentiator
- **Article:** [Claude Managed Agents](/2026/04/10/claude-managed-agents-codex-cli/) — this earlier article anticipated managed agents; Routines are the shipped version and should be cross-referenced
- **Article:** [Desktop Superapp War](/2026/04/14/desktop-superapp-war-codex-scratchpad-vs-claude-epitaxy/) — the desktop redesign announced alongside Routines directly relates to this analysis

---

## Technical Details

### API Trigger Example

```bash
curl -X POST https://api.anthropic.com/v1/claude_code/routines/trig_01ABCDEFGHJKLMNOPQRSTUVW/fire \
  -H "Authorization: Bearer sk-ant-oat01-xxxxx" \
  -H "anthropic-beta: experimental-cc-routine-2026-04-01" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"text": "Sentry alert SEN-4521 fired in prod. Stack trace attached."}'
```

Response:

```json
{
  "type": "routine_fire",
  "claude_code_session_id": "session_01HJKLMNOPQRSTUVWXYZ",
  "claude_code_session_url": "https://claude.ai/code/session_01HJKLMNOPQRSTUVWXYZ"
}
```

### CLI Commands

- `/schedule` — create a scheduled routine conversationally
- `/schedule daily PR review at 9am` — create with inline description
- `/schedule list` — list all routines
- `/schedule update` — modify an existing routine
- `/schedule run` — trigger a routine immediately

### GitHub Trigger Filter Operators

equals, contains, starts with, is one of, is not one of, matches regex

### Supported GitHub Events

| Event | Triggers when |
|-------|--------------|
| Pull request | Opened, closed, assigned, labeled, synchronized, or otherwise updated |
| Release | Created, published, edited, or deleted |

---

## Personal Notes

Major competitive move from Anthropic. This positions Claude Code as the first CLI-rooted tool with cloud-hosted autonomous execution — a category Codex CLI has not entered yet (though Goal Mode PRs in codex-rs suggest OpenAI is heading there). The combination of schedule + API + GitHub triggers with MCP connectors makes Routines a genuine workflow automation platform, not just a coding agent feature. The 5/15/25 daily limits are generous enough for individual developers and small teams, but enterprise customers will likely need the overage billing option.

The research preview status is worth monitoring — the beta header versioning scheme (`experimental-cc-routine-2026-04-01`) suggests Anthropic expects breaking changes, and the per-routine hourly webhook caps during preview could limit GitHub-triggered use cases for high-activity repositories.

For the book, this changes the competitive landscape chapter significantly. The narrative was "Claude Code competes on model quality; Codex CLI competes on openness and extensibility." Now it is "Claude Code competes on model quality *and* cloud-native autonomous execution; Codex CLI competes on openness, extensibility, and local control." That is a harder competitive position to defend.
