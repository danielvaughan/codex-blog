---
title: "Codex CLI in 2026: What's New, What's Changed and What's Coming"
date: 2026-03-27T09:00:00+00:00
description: "A comprehensive guide to every major Codex CLI change in 2026 — new models, hooks, subagents, Codex Spark, enterprise features, and what's still on the roadmap."
tags:
  - opinion
  - product-direction
  - subagents
  - hooks
  - enterprise
  - codex-cli
  - changelog
  - models
---
![Sketchnote diagram for: Codex CLI in 2026: What's New, What's Changed and What's Coming](/sketchnotes/articles/2026-03-27-codex-cli-in-2026-whats-new.png)

*Written 2026-03-27. Based on official changelog, GitHub releases, and community research.*

---

Codex CLI has changed substantially since its initial release. This article captures the most important developments of 2026 so far — from new models to enterprise hooks, from the subagents GA to the upcoming plugin system — in one place.

---

## The Big Picture

Three themes dominate 2026 Codex development:

1. **Speed and real-time interaction** — GPT-5.3-Codex-Spark (1,000+ tok/s), WebSocket latency improvements
2. **Agentic autonomy** — Subagents GA, Smart Approvals, PostToolUse hooks, worktree automations
3. **Enterprise readiness** — Custom CA certs, userpromptsubmit hooks, managed configs, plugin policy enforcement

---

## New Models

### `gpt-5-codex` and `gpt-5-codex-mini` (Late March 2026)

The new default models for Codex CLI, announced by @thsottiaux:

- **`gpt-5-codex`** — new flagship: *"fast for small things, working hard when it matters — solid jump in produced code quality"*
- **`gpt-5-codex-mini`** — up to **4x more usage included** for Plus/Edu/Team plans; 50% increased limits overall; Pro users get priority processing

```bash
codex -m gpt-5-codex        # flagship
codex -m gpt-5-codex-mini   # usage-efficient for subagents and exploration
```

**Recommendation:** `gpt-5-codex` for primary work; `gpt-5-codex-mini` for subagents, exploration, and large-file review where token efficiency matters.

---

### GPT-5.4 and GPT-5.4-mini (March 17, 2026)

- **GPT-5.4** — frontier reasoning + coding model, was the previous default
- **GPT-5.4-mini** — 2x faster than GPT-5.4, uses 30% of included limits; good for fast iteration and subagent work

---

### GPT-5.3-Codex-Spark (February 12, 2026) ⭐

OpenAI's most novel model release: purpose-built for **real-time coding**.

| Spec | Value |
|------|-------|
| Speed | **1,000+ tokens/second** |
| Context | 128k tokens |
| Input types | Text only (at launch) |
| Hardware | Cerebras Wafer Scale Engine 3 |
| Availability | Research preview — ChatGPT Pro only |
| API | Not available at launch |

**Why it matters:** Codex-Spark makes the agent feel like a live collaborator, not a batch process. OpenAI also reduced full pipeline latency alongside the model — per-token overhead down 30%, time-to-first-token down 50%, per-round-trip overhead down 80% via persistent WebSocket.

**Strategic significance:** This is the first OpenAI model deployed on Cerebras hardware (not NVIDIA GPUs) — part of a 10B+ partnership announced in early 2026. It signals OpenAI investing in diverse inference infrastructure for ultra-low-latency AI.

**The two-mode vision:** With Spark + GPT-5-codex, Codex now supports both *long-running autonomous tasks* (high-reasoning models) and *real-time pair-programming* (Spark). Different tools for different moments in the development loop.

---

## Subagents: GA Since March 16, 2026 ⭐

Subagents shipped as generally available in **v0.115.0** (March 16), retweeted by Sam Altman. Up to **6 subagents** can run concurrently.

### Built-in roles

| Role | Access | Best for |
|------|--------|----------|
| `explorer` | Read-only | Codebase scanning, dependency mapping, context gathering |
| `worker` | Read-write | Implementation, large batches of small tasks |
| `default` | Read-write | General-purpose single task |

### Custom agents (TOML)

Define in `~/.codex/agents/<name>.toml` (personal) or `.codex/agents/<name>.toml` (project):

```toml
name = "test-runner"
description = "Runs the test suite and diagnoses failures"
developer_instructions = "Focus only on test analysis and fixes."
model = "gpt-5-codex-mini"
reasoning_effort = "medium"
```

### Smart Approvals (v0.115.0)

Rather than blindly auto-approving in `--full-auto` mode, a lightweight **guardian subagent** reviews pending actions and routes them — approve silently, escalate to user, or block. Makes full-auto safer for CI and overnight runs.

---

## Hooks: Three Live, One Coming

### userpromptsubmit (v0.116.0, March 19)

Fires **before** a prompt enters conversation history. Key for audit logging and policy enforcement in regulated environments:

```toml
# ~/.codex/config.toml
[hooks]
userpromptsubmit = "~/.codex/hooks/audit-prompt.sh"
```

The hook receives the raw prompt on stdin. Exit 0 = allow, exit 1 = block. The hook can also modify the prompt before passing it on.

### PostToolUse (v0.117.0, arriving soon)

Fires **after** any tool executes. Key use case: auto-run tests after every file write.

```toml
[hooks]
post_tool_use = "~/.codex/hooks/auto-test.sh"
```

```bash
# auto-test.sh — receives tool event JSON on stdin
TOOL=$(cat | jq -r '.tool_name')
if [[ "$TOOL" == "write_file" || "$TOOL" == "edit_file" ]]; then
  npm test --silent 2>&1 | tail -5
fi
```

Note: PostToolUse fires *after* the tool has run, so it cannot undo. For blocking behaviour use `userpromptsubmit`.

### Existing hooks (stable)

| Hook | Trigger |
|------|---------|
| `SessionStart` | Session begins — load context, pre-flight checks |
| `Stop` | Session ends — cleanup, notify team, audit log |

---

## New CLI Features

### Integrated Terminal Reading (late March 2026)

Codex can now **read the integrated terminal** for the current thread. This means:
- It can check whether your dev server is running
- It can see failed build output without you pasting it in
- The agent maintains awareness of your terminal state while working

This closes a significant gap: previously, you had to manually copy-paste terminal output to give Codex context about runtime errors or running processes.

### `codex cloud` Command

New command for working with cloud-based tasks from the terminal:

```bash
codex cloud   # triage and launch Codex cloud tasks without leaving terminal
```

Lets you launch cloud tasks, choose environments, and apply resulting diffs — all from within your terminal session.

### Mid-Thread Conversation Forks

You can now fork a conversation from **any earlier message**, not just the latest turn. Useful when you want to explore an alternative approach from a previous decision point without losing the current thread.

```bash
/fork   # fork from any earlier message in the current thread
```

---

## Enterprise Features

### Custom CA Certificate Support (v0.116.0)

Corporate environments with TLS inspection proxies (Zscaler, etc.) can now configure custom CA certificates:

```bash
export SSL_CERT_FILE=/path/to/corporate-ca.pem
# Or use the Codex-specific env vars
```

This unblocks Codex in the most common enterprise network configurations.

### Device-Code ChatGPT Sign-In (v0.116.0)

Onboarding now supports ChatGPT account login directly — no separate API key required for Plus/Pro users. Token refresh is handled automatically for existing sessions.

```bash
brew install codex
codex   # follow device-code sign-in flow
```

### Plugin Policy Enforcement (v0.116.0+)

Enterprise admins can configure a **plugin suggestion allowlist** — Codex will only prompt to install approved plugins. Install/uninstall state syncs remotely, making fleet management feasible.

---

## Automations: Significantly Upgraded (March 12–27)

The Codex app's Automations feature received a major update:

- **Local or worktree execution** — choose per automation; worktree-based for cleaner isolation
- **Custom reasoning levels and models** — e.g., light reasoning for triage, high for complex analysis
- **Template gallery** — built-in templates as starting points
- **Custom themes** — accent/background/foreground, UI and code fonts; shareable

---

## v0.117.0 Alpha: What's Coming

Active development as of late March 2026 (22+ alpha releases in 2 days):

- **PostToolUse hook** — see above
- **Plugin system GA** — plugins becoming first-class feature
- **Improved MCP tool elicitation** — better UX for MCP tool selection
- **Additional app-server transports** — beyond stdio; WebSocket auth for app-server
- **MCP tool call span instrumentation** — tracing/observability for MCP calls
- **Home-relative paths on Windows** — `~/` paths work correctly
- **Skills extracted to separate crate** — architectural improvement

---

## Ecosystem Growth

The broader Codex ecosystem has matured considerably:

- **awesome-codex-cli** now lists 60+ curated tools
- **Agentmaxxing** has become a mainstream pattern — multiplexers (AMUX, dmux, Emdash) make running 10+ parallel agents practical
- **OpenAI acquired Astral** (uv, ruff) — signal of deeper Python tooling investment
- **opencode** reached 95K stars as the leading open-source alternative
- **Nimbalyst** replaced Crystal as the leading visual workspace for parallel sessions

---

## What's Still Coming

| Feature | Status | Why it matters |
|---------|--------|---------------|
| Codex-Spark GA | Research preview | When GA, fastest feedback loop for everyday coding |
| PostToolUse hook stable | v0.117.0 alpha | Auto-test-on-write without human approval |
| Plugin system GA | v0.117.0 alpha | First-class extensibility |
| Image output on CLI | Cloud only | Visual UI iteration from terminal |
| Windows native (non-WSL) | Experimental | Unblocks Windows-first enterprise teams |
| More SDK languages | Python shipping | Embed Codex in CI, custom tooling |

---

## The Version Trail

| Version | Date | Key features |
|---------|------|-------------|
| v0.117.0 | Coming soon | PostToolUse hook, Plugin GA, WebSocket auth |
| v0.116.0 | March 19 | userpromptsubmit hook, Custom CA certs, ChatGPT login |
| v0.115.0 | March 16 | Subagents GA, Smart Approvals, Python SDK |
| v0.114.0 | March 11 | Hooks engine, experimental code mode, health checks |

---

## TL;DR

If you last looked at Codex CLI a few months ago, here's what changed most:

1. **New models**: `gpt-5-codex` + `gpt-5-codex-mini` are the new defaults. Codex-Spark adds real-time speed for Pro users.
2. **Subagents are GA**: 6 concurrent agents, guardian-based Smart Approvals for safe full-auto
3. **Hooks are real**: `userpromptsubmit` for enterprise audit trails; `PostToolUse` landing soon
4. **Terminal awareness**: Codex can now read your terminal state to stay context-aware
5. **Enterprise is serious**: Custom CA certs, plugin policy, device-code auth — corporate deployment is now tractable
6. **The ecosystem**: 60+ community tools, multiplexers for 10+ parallel agents, agentmaxxing as a mainstream practice
