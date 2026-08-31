---
title: "Codex CLI Doctor: Diagnostics, Troubleshooting, and Support-Ready Reports"
type: Technical Article
timestamp: 2026-06-02T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-06-02-codex-cli-doctor-diagnostics-troubleshooting-support-workflows"
tags: ["codex-cli", "diagnostics", "troubleshooting", "codex-doctor", "support", "debugging", "v0.135", "v0.136"]
date: 2026-06-02T09:00:00+00:00
last_modified_at: 2026-08-31T08:16:21+01:00
---
# Codex CLI Doctor: Diagnostics, Troubleshooting, and Support-Ready Reports


When a coding agent misbehaves, the instinct is to restart and hope for the best. Codex CLI's `codex doctor` command offers a better path: a structured diagnostic report that examines installation integrity, authentication, terminal compatibility, network connectivity, and local state — all in one invocation. Introduced in v0.131.0 and substantially expanded in v0.135.0[^1], it has quickly become the first command you should run when something goes wrong.

This article covers the command's diagnostic categories, output formats, integration with OpenAI's support pipeline, and practical troubleshooting workflows for the failure patterns that surface most often.

## What `codex doctor` Actually Checks

The command organises its checks into six sections, each targeting a distinct failure domain[^2]:

### Diagnostic Categories

| Category | What It Examines |
|---|---|
| **Environment** | Runtime provenance, install consistency, bundled/system search readiness, terminal/multiplexer metadata (TERM, locale, size, TERMINFO, tmux, zellij), SQLite integrity for state and log databases[^3] |
| **Configuration** | `config.toml` parse status, model/provider settings, auth modes, MCP server definitions, feature flags, sandbox policies[^2] |
| **Updates** | Update configuration consistency, available versions, update-target mismatches[^3] |
| **Connectivity** | Network environment, WebSocket status, provider-aware HTTP endpoint reachability (including Bedrock region fallback as of v0.136.0)[^4] |
| **Background Server** | App-server daemon state, remote connection details, server version[^1] |
| **Notes** | Promoted anomalies — available updates, large rollout directories, auth conflicts, mixed auth signals[^3] |

Each check emits a status indicator: `✓ ok`, `⚠ warn`, `✗ fail`, or `○ idle`[^3].

## Command Flags

```bash
# Full detailed report (default)
codex doctor

# Compact grouped summary
codex doctor --summary

# Machine-readable JSON for support tickets
codex doctor --json

# Expand truncated lists
codex doctor --all

# ASCII-only output (for terminals without Unicode)
codex doctor --ascii

# Strip ANSI colour codes
codex doctor --no-color
```

The flags are composable. For a CI health check, `codex doctor --json --no-color` pipes cleanly into `jq` or a monitoring agent[^5].

## Output Formats

### Human-Readable (Default)

The default output groups checks by category with status indicators and summary counts:

```
Environment
  ✓ runtime         Codex 0.136.0 (rust, installed via npm)
  ✓ terminal        xterm-256color, 120×40, tmux 3.4
  ✓ sqlite          State and log databases intact
  ⚠ rollout-dir     427 sessions, 1.2 GB — consider archiving

Configuration
  ✓ config-parse    config.toml valid
  ✓ auth            API key (OPENAI_API_KEY)
  ✓ mcp-servers     3 configured, 3 reachable
  ✗ feature-flags   Unknown flag 'beta_goal_v2' — check for typos

Connectivity
  ✓ api-endpoint    api.openai.com reachable (142ms)
  ✓ websocket       wss://… connected (89ms)

──────────────────────────────
✓ 8 ok  ⚠ 1 warn  ✗ 1 fail
```

### JSON Schema

The `--json` flag emits a structured, redacted report suitable for support tooling[^3]:

```json
{
  "schema_version": 1,
  "codex_version": "0.136.0",
  "overall_status": "fail",
  "checks": {
    "runtime": {
      "status": "ok",
      "summary": "Codex 0.136.0 (rust, installed via npm)",
      "details": {
        "version": "0.136.0",
        "install_method": "npm",
        "binary_path": "/usr/local/bin/codex"
      }
    },
    "feature-flags": {
      "status": "fail",
      "summary": "Unknown flag 'beta_goal_v2'",
      "details": {
        "unknown_flags": ["beta_goal_v2"],
        "active_flags": ["session_archive", "osc8_links"]
      }
    }
  }
}
```

Sensitive values — API keys, tokens, internal paths — are automatically redacted before serialisation[^3].

## Integration with the Support Pipeline

The doctor report is not just a local convenience. It plugs directly into OpenAI's support workflow in three ways:

1. **Bug template**: The CLI issue template on GitHub requests `codex doctor --json` output as a required field[^3]
2. **Feedback uploads**: When you run `/feedback` inside a TUI session, the doctor report auto-attaches as `codex-doctor-report.json`[^3]
3. **Sentry tags**: Doctor results generate Sentry tags reflecting `overall_status` and failing check IDs, enabling OpenAI to correlate crash reports with environment anomalies[^3]

```mermaid
flowchart LR
    A[User hits issue] --> B[codex doctor --json]
    B --> C{File support ticket?}
    C -->|GitHub Issue| D[Paste JSON in bug template]
    C -->|In-session| E[/feedback auto-attaches report]
    D --> F[OpenAI support triage]
    E --> F
    F --> G[Sentry tags from doctor checks]
```

## Companion Diagnostic Commands

`codex doctor` sits within a broader diagnostic toolkit[^5][^6]:

| Command | Purpose |
|---|---|
| `codex doctor` | Full environment diagnostic report |
| `codex debug models` | Dump the recognised model catalogue as JSON |
| `codex debug models --bundled` | Print only bundled models (no remote fetch) |
| `codex debug app-server send-message-v2` | Test app-server V2 protocol flow |
| `/status` | In-TUI: token usage, context window percentage, remote connection details |
| `/config` | In-TUI: effective configuration values and their source files |
| `/mcp verbose` | In-TUI: full MCP server diagnostics with timeouts and auth errors |
| `/feedback` | In-TUI: submit logs and doctor report to OpenAI |

## Troubleshooting the Four Most Common Failures

Based on community reports and the patterns the doctor command is designed to detect[^7], here are the workflows for the most frequent issues.

### 1. Authentication Failures (401 Unauthorised)

Symptoms: `✗ auth` in doctor output, "401 Unauthorized" in `/feedback`.

```bash
# Check current auth state
codex doctor --json | jq '.checks.auth'

# If using API key
echo $OPENAI_API_KEY | head -c 8  # verify prefix

# If using ChatGPT browser auth — tokens may have expired
codex login status
codex login  # re-authenticate
```

The doctor command detects "mixed auth signals" — for instance, both `OPENAI_API_KEY` and a cached browser token present — which is a common source of 401 errors after switching auth methods[^3].

### 2. MCP Server Connection Failures

Symptoms: `✗ mcp-servers` in doctor output, "MCP handshake timeout" in logs.

```bash
# Detailed MCP diagnostics
codex doctor --all  # expands per-server details

# In TUI, for live diagnostics
/mcp verbose
```

As of v0.134.0, MCP setup supports per-server environment targeting and OAuth for streamable HTTP servers[^4]. If a server was configured before this release, its environment block may need updating. The doctor report surfaces which servers are unreachable and why.

### 3. Terminal Rendering Corruption

Symptoms: `⚠ terminal` in doctor output, garbled TUI display.

```bash
codex doctor --json | jq '.checks.terminal.details'
```

The terminal check captures TERM, locale, terminal size, TERMINFO path, and multiplexer state (tmux/zellij)[^3]. Common culprits:

- **tmux with wrong TERM**: set `TERM=xterm-256color` inside tmux
- **Zellij stderr overlap**: fixed in v0.135.0 but requires the latest Zellij plugin[^1]
- **Windows VT mode**: v0.134.0 fixed virtual terminal mode restoration before drawing[^4]

### 4. App-Server Disconnection Loops

Symptoms: `✗ app-server` in doctor output, WebSocket closure code 1006.

```bash
# Check daemon state
codex doctor --json | jq '.checks["app-server"]'

# Manual restart
codex app-server stop
codex app-server start
```

The v0.134.0 release addressed stale exec-server WebSocket reconnection with auth recovery retries and stream retries[^4]. If the problem persists, `codex debug app-server send-message-v2 "test"` exercises the full V2 protocol flow to isolate whether the issue is auth, network, or daemon state[^5].

## Automating Health Checks

For teams running Codex in CI or shared environments, a periodic health check prevents silent degradation:

```bash
#!/usr/bin/env bash
# codex-health-check.sh — run from cron or CI

report=$(codex doctor --json 2>/dev/null)
status=$(echo "$report" | jq -r '.overall_status')

if [ "$status" != "ok" ]; then
  failing=$(echo "$report" | jq -r '
    [.checks | to_entries[] | select(.value.status == "fail") | .key]
    | join(", ")
  ')
  echo "::error::Codex doctor failed: $failing"
  echo "$report" > codex-doctor-report.json
  exit 1
fi

echo "Codex environment healthy"
```

For enterprise deployments using `requirements.toml` managed policies, the doctor report can validate that local installations conform to organisational baselines before granting sandbox access[^8].

## When Not to Use `codex doctor`

The command examines the local environment and configuration. It does not diagnose:

- **Model behaviour issues** (hallucinations, incorrect code) — use evals and `/feedback` with a Request ID
- **Rate limiting or quota exhaustion** — check `/status` for token usage or the OpenAI dashboard
- **Remote Codex cloud agent failures** — these run in OpenAI's infrastructure, not locally

For model-level issues, the Request ID from `/feedback` is more useful than the doctor report, as it lets OpenAI trace server-side logs[^7].

## Summary

`codex doctor` transforms Codex CLI troubleshooting from guesswork into structured diagnosis. Run it first when something breaks, attach `--json` output to support tickets, and automate it in CI to catch environment drift before it causes failures. Combined with `/status`, `/mcp verbose`, and `/feedback`, it forms a complete observability stack for the local agent runtime.

## Citations

[^1]: [Codex CLI v0.135.0 Release Notes](https://github.com/openai/codex/releases/tag/rust-v0.135.0) — GitHub, May 2026
[^2]: [Codex CLI Command Line Reference — `codex doctor`](https://developers.openai.com/codex/cli/reference) — OpenAI Developers, June 2026
[^3]: [feat(cli): add codex doctor diagnostics — PR #22336](https://github.com/openai/codex/pull/22336) — openai/codex, GitHub, 2026
[^4]: [Codex Changelog](https://developers.openai.com/codex/changelog) — OpenAI Developers, June 2026
[^5]: [Codex CLI v0.135 Reference: history search, doctor, profiles](https://blakecrosley.com/guides/codex) — Blake Crosley, May 2026
[^6]: [Codex CLI Logs: Location, Debug Flags & 401 Error Fix](https://smartscope.blog/en/generative-ai/chatgpt/codex-cli-diagnostic-logs-deep-dive/) — SmartScope, 2026
[^7]: [Codex CLI Commands List 2026 — Free Cheat Sheet](https://toolsbase.dev/en/reference/codex-commands) — Toolsbase, 2026
[^8]: [Codex CLI Enterprise Deployment: Managed Configuration, RBAC, and the GitHub Enterprise Server Connector](https://codex.danielvaughan.com/2026/06/01/codex-cli-enterprise-deployment/) — Codex Knowledge Base, June 2026
