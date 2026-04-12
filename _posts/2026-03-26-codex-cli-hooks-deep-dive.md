---
title: "Codex CLI Hooks Deep Dive: SessionStart, Stop and UserPromptSubmit"
date: 2026-03-26T09:00:00+00:00
tags:
  - codex-cli
  - hooks
  - automation
  - session-management
  - security
  - notifications
  - secrets
  - developer-experience
---

![Sketchnote diagram for: Codex CLI Hooks Deep Dive: SessionStart, Stop and UserPromptSubmit](/sketchnotes/articles/2026-03-26-codex-cli-hooks-deep-dive.png)

# Codex CLI Hooks Deep Dive: SessionStart, Stop and UserPromptSubmit


---

Codex CLI v0.116.0 shipped the first three hook events: `SessionStart`, `Stop`, and `UserPromptSubmit`. These are session-scoped hooks that fire at the boundaries of an agent session rather than around individual tool calls. They give you the ability to inject context when a session begins, react when the agent finishes a turn, and inspect or redact prompts before they enter conversation history.

This article is a deep dive into all three events. We cover the configuration format, the JSON protocol each hook receives on stdin, the exit-code semantics that control blocking, and practical production patterns: injecting branch and ticket context on session start, sending desktop notifications when the agent stops, and redacting secrets from prompts before they reach the model.

> **Note:** Tool-level hooks (`PreToolUse` and `PostToolUse`) shipped in v0.117.0 and are covered in the companion article: [Codex CLI PreToolUse & PostToolUse Hooks](/2026/03/31/codex-cli-pretooluse-posttooluse-hooks/).

---

## Table of Contents

1. [Enabling Hooks](#enabling-hooks)
2. [Hook Configuration Format](#hook-configuration-format)
3. [How Hooks Execute](#how-hooks-execute)
4. [SessionStart — Inject Context When the Session Opens](#sessionstart--inject-context-when-the-session-opens)
5. [Stop — React When the Agent Finishes a Turn](#stop--react-when-the-agent-finishes-a-turn)
6. [UserPromptSubmit — Inspect and Redact Prompts](#userpromptsubmit--inspect-and-redact-prompts)
7. [Layering Hooks: Global and Repo-Level](#layering-hooks-global-and-repo-level)
8. [Exit Code Semantics](#exit-code-semantics)
9. [Practical Example: Full hooks.json for a Team](#practical-example-full-hooksjson-for-a-team)
10. [Debugging Hooks](#debugging-hooks)
11. [Limitations](#limitations)
12. [References](#references)

---

## Enabling Hooks

Hooks are gated behind a feature flag. Add the following to your Codex configuration file:

```toml
# ~/.codex/config.toml
[features]
codex_hooks = true
```

Without this flag, Codex ignores all `hooks.json` files. The flag exists because hooks run arbitrary shell commands outside the sandbox — enabling them is an explicit opt-in to that trust model.

> Hooks are **disabled on Windows** as of v0.116.0. The hook engine relies on Unix process semantics (stdin piping, exit codes, signal handling) that are not yet ported to the Windows experimental build.

---

## Hook Configuration Format

Hooks are defined in a JSON file named `hooks.json`, placed in one of two locations:

- **User-level (global):** `~/.codex/hooks.json`
- **Repo-level:** `<repo-root>/.codex/hooks.json`

Both files are loaded and merged. Hooks from both locations run concurrently when they match the same event — the repo-level file does not override or replace the global file.

The top-level structure is:

```json
{
  "hooks": {
    "<EventName>": [
      {
        "matcher": "<optional-matcher-string>",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/your/script.sh",
            "timeout": 10,
            "statusMessage": "Running my hook..."
          }
        ]
      }
    ]
  }
}
```

### Field Reference

| Field | Required | Description |
|-------|----------|-------------|
| `type` | Yes | Always `"command"` in v0.116.0 |
| `command` | Yes | Path to an executable script or inline shell command |
| `timeout` | No | Maximum seconds the hook may run before being killed (default: 30) |
| `statusMessage` | No | Message shown in the TUI while the hook runs |
| `matcher` | No | Regex string to narrow when the hook fires (event-specific) |

The `matcher` field is only meaningful for events that define a matcher dimension. For `SessionStart`, the matcher tests against the `source` field (`"startup"` or `"resume"`). For `UserPromptSubmit` and `Stop`, there is no matcher — hooks fire unconditionally for every occurrence.

---

## How Hooks Execute

When an event fires, Codex:

1. Collects all matching hook entries from all loaded `hooks.json` files
2. Spawns each hook's command as a child process
3. Pipes a JSON payload to the process's **stdin**
4. Reads **stdout** for structured JSON responses
5. Reads **stderr** for human-readable messages (displayed in the TUI)
6. Interprets the **exit code** to determine the hook's decision

Multiple matching hooks for the same event run **concurrently**. One hook cannot prevent another from starting. This means you cannot build a "first hook decides, others skip" pattern — all hooks execute and all their results are collected.

The JSON payload on stdin always includes these common fields:

```json
{
  "session_id": "019d0352-a1b7-7c8a-b123-456789abcdef",
  "hook_event_name": "<EventName>",
  "cwd": "/workspace/my-project",
  "model": "gpt-5.4"
}
```

Additional fields are event-specific and documented in each section below.

---

## SessionStart — Inject Context When the Session Opens

`SessionStart` fires once when a Codex session begins. It has two modes, distinguished by the `source` field:

- `"startup"` — a brand-new session
- `"resume"` — resuming a previously saved session

### JSON Protocol (stdin)

```json
{
  "session_id": "019d0352-a1b7-7c8a-b123-456789abcdef",
  "hook_event_name": "SessionStart",
  "source": "startup",
  "cwd": "/workspace/my-project",
  "model": "gpt-5.4"
}
```

### Matcher

The matcher for `SessionStart` tests against the `source` field. Use `"startup"` to run only on fresh sessions, `"resume"` for resumed sessions, or omit the matcher to run on both.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "~/.codex/hooks/inject-context.sh",
            "statusMessage": "Injecting project context..."
          }
        ]
      }
    ]
  }
}
```

### Practical Example: Inject Branch and Ticket Context

One of the most valuable `SessionStart` patterns is injecting the current Git branch name and any associated ticket information into the session. This gives the agent awareness of what you are working on without you needing to explain it every time.

```bash
#!/usr/bin/env bash
# ~/.codex/hooks/inject-context.sh
# Injects Git branch, recent commits, and ticket context into the session.

set -euo pipefail

# Read the JSON payload (required even if we don't use all fields)
INPUT=$(cat)
CWD=$(echo "$INPUT" | jq -r '.cwd // "."')

cd "$CWD" 2>/dev/null || exit 0

# Bail if not in a git repo
if ! git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  exit 0
fi

BRANCH=$(git symbolic-ref --short HEAD 2>/dev/null || echo "detached")
RECENT_COMMITS=$(git log --oneline -5 2>/dev/null || echo "no commits")
DIRTY_FILES=$(git diff --name-only 2>/dev/null | head -20)

# Build context message
CONTEXT="## Session Context (auto-injected)
- **Branch:** \`${BRANCH}\`
- **Recent commits:**
\`\`\`
${RECENT_COMMITS}
\`\`\`"

# If the branch name contains a ticket reference (e.g., PROJ-1234), add it
TICKET=$(echo "$BRANCH" | grep -oE '[A-Z]+-[0-9]+' | head -1)
if [ -n "$TICKET" ]; then
  CONTEXT="${CONTEXT}
- **Ticket:** ${TICKET}"

  # If you have a Jira/Linear CLI, pull the ticket title
  if command -v jira >/dev/null 2>&1; then
    TITLE=$(jira issue view "$TICKET" --plain 2>/dev/null | head -1 || echo "")
    if [ -n "$TITLE" ]; then
      CONTEXT="${CONTEXT} — ${TITLE}"
    fi
  fi
fi

if [ -n "$DIRTY_FILES" ]; then
  CONTEXT="${CONTEXT}
- **Uncommitted changes:**
\`\`\`
${DIRTY_FILES}
\`\`\`"
fi

# Output as JSON — Codex reads stdout for structured hook responses
echo "$CONTEXT" | jq -Rs '{additionalContext: .}'

exit 0
```

When this hook runs, the agent sees the branch name, recent commits, and ticket reference as injected context at the start of the session. This eliminates the "what branch am I on?" back-and-forth.

### Practical Example: Record Session Start for Audit

For teams that need to track agent usage, log every session start:

```bash
#!/usr/bin/env bash
# ~/.codex/hooks/log-session-start.sh

INPUT=$(cat)
SESSION_ID=$(echo "$INPUT" | jq -r '.session_id // "unknown"')
SOURCE=$(echo "$INPUT" | jq -r '.source // "unknown"')
MODEL=$(echo "$INPUT" | jq -r '.model // "unknown"')
CWD=$(echo "$INPUT" | jq -r '.cwd // "unknown"')

LOG_DIR="$HOME/.codex/session-logs"
mkdir -p "$LOG_DIR"

echo "{\"event\":\"session_start\",\"session_id\":\"${SESSION_ID}\",\"source\":\"${SOURCE}\",\"model\":\"${MODEL}\",\"cwd\":\"${CWD}\",\"timestamp\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",\"user\":\"$(whoami)\"}" \
  >> "$LOG_DIR/sessions.jsonl"

exit 0
```

---

## Stop — React When the Agent Finishes a Turn

`Stop` fires when the agent's turn completes — all tool calls are done and the final response is ready. This is the event to use for notifications, progress logging, and auto-continuation logic.

### JSON Protocol (stdin)

```json
{
  "session_id": "019d0352-a1b7-7c8a-b123-456789abcdef",
  "hook_event_name": "Stop",
  "turn_id": "019d036d-c7fa-72d2-b6fd-78878bfe34e4",
  "cwd": "/workspace/my-project",
  "model": "gpt-5.4"
}
```

The `turn_id` field was added in PR #15118 (merged in v0.116.0, March 19, 2026). It lets you correlate hook invocations with specific conversation turns in your audit logs.

### Matcher

`Stop` has no matcher field. The hook fires for every turn completion.

### Practical Example: Desktop Notifications

When running Codex on long tasks, you want to know when it finishes so you can review the output. Desktop notifications solve this.

**macOS (osascript):**

```bash
#!/usr/bin/env bash
# ~/.codex/hooks/notify-stop-macos.sh
# Desktop notification when Codex finishes a turn.

INPUT=$(cat)
SESSION_ID=$(echo "$INPUT" | jq -r '.session_id // "unknown"' | cut -c1-8)

osascript -e "display notification \"Turn complete (session ${SESSION_ID}...)\" with title \"Codex CLI\" sound name \"Glass\""

exit 0
```

**Linux (notify-send):**

```bash
#!/usr/bin/env bash
# ~/.codex/hooks/notify-stop-linux.sh

INPUT=$(cat)
SESSION_ID=$(echo "$INPUT" | jq -r '.session_id // "unknown"' | cut -c1-8)

notify-send "Codex CLI" "Turn complete (session ${SESSION_ID}...)" \
  --icon=terminal \
  --urgency=normal \
  --expire-time=5000

exit 0
```

**Cross-platform (terminal bell):**

```bash
#!/usr/bin/env bash
# Works in any terminal emulator. Simple but effective.
printf '\a'
exit 0
```

### Practical Example: Slack Notification via Webhook

For team visibility, post to a Slack channel when the agent finishes:

```bash
#!/usr/bin/env bash
# ~/.codex/hooks/notify-stop-slack.sh

INPUT=$(cat)
SESSION_ID=$(echo "$INPUT" | jq -r '.session_id // "unknown"' | cut -c1-8)
CWD=$(echo "$INPUT" | jq -r '.cwd // "unknown"')
MODEL=$(echo "$INPUT" | jq -r '.model // "unknown"')

WEBHOOK_URL="${CODEX_SLACK_WEBHOOK:-}"
if [ -z "$WEBHOOK_URL" ]; then
  exit 0
fi

PAYLOAD=$(jq -n \
  --arg text ":robot_face: Codex turn complete\n*Session:* ${SESSION_ID}...\n*Dir:* ${CWD}\n*Model:* ${MODEL}\n*User:* $(whoami)" \
  '{text: $text}')

curl -s -X POST -H 'Content-Type: application/json' \
  -d "$PAYLOAD" "$WEBHOOK_URL" >/dev/null 2>&1 &

exit 0
```

### Practical Example: Auto-Continue on Truncated Output

Sometimes the agent stops mid-output due to token limits. You can detect this and signal that continuation is needed:

```bash
#!/usr/bin/env bash
# ~/.codex/hooks/auto-continue.sh
# Detects truncated output and writes a continuation signal.

INPUT=$(cat)
SESSION_ID=$(echo "$INPUT" | jq -r '.session_id // "unknown"')

# Check the last output file for truncation markers
LAST_OUTPUT="/tmp/codex-last-output-${SESSION_ID}.txt"
if [ -f "$LAST_OUTPUT" ]; then
  LAST_LINE=$(tail -1 "$LAST_OUTPUT")
  if echo "$LAST_LINE" | grep -qE '^\.\.\.$|to be continued|truncated|…$'; then
    echo "Output appears truncated — consider sending 'continue'" >&2
  fi
fi

exit 0
```

---

## UserPromptSubmit — Inspect and Redact Prompts

`UserPromptSubmit` fires after the user submits a prompt but **before** it enters conversation history and before the model processes it. This is the right place for:

- Redacting secrets (API keys, tokens, passwords) before they reach the model
- Blocking prompts that match a content policy
- Audit logging of all prompts for compliance

### JSON Protocol (stdin)

```json
{
  "session_id": "019d0352-a1b7-7c8a-b123-456789abcdef",
  "hook_event_name": "UserPromptSubmit",
  "prompt": "Use the API key sk-proj-abc123def456 to call the endpoint",
  "turn_id": "019d036d-c7fa-72d2-b6fd-78878bfe34e4",
  "cwd": "/workspace/my-project",
  "model": "gpt-5.4"
}
```

The `prompt` field contains the full text of the user's prompt. The `turn_id` field (added in PR #15118, v0.116.0) correlates this prompt with its turn in the conversation.

### Matcher

`UserPromptSubmit` has no matcher field. The hook fires for every prompt submission.

### Blocking a Prompt

To block a prompt, exit with code 2 and write the reason to stderr:

```bash
#!/usr/bin/env bash
# Block prompts containing destructive instructions
INPUT=$(cat)
PROMPT=$(echo "$INPUT" | jq -r '.prompt // ""')

if echo "$PROMPT" | grep -qiE 'delete all|drop (all )?table|rm -rf /'; then
  echo "Blocked: prompt contains potentially destructive instructions" >&2
  exit 2
fi

exit 0
```

When a prompt is blocked, the user sees the stderr message in the TUI and can rephrase.

### Practical Example: Redact Secrets Before They Reach the Model

This is the highest-value `UserPromptSubmit` pattern. Users frequently paste configuration snippets, environment variables, or log output that contains API keys, tokens, or passwords. This hook catches common secret patterns and redacts them before the prompt enters conversation history.

**Bash version:**

```bash
#!/usr/bin/env bash
# ~/.codex/hooks/redact-secrets.sh
# Scans the user prompt for secret patterns and warns (or blocks).

INPUT=$(cat)
PROMPT=$(echo "$INPUT" | jq -r '.prompt // ""')

# Patterns that look like secrets
FOUND=0

# OpenAI API keys
if echo "$PROMPT" | grep -qE 'sk-[A-Za-z0-9]{20,}'; then
  echo "WARNING: Detected OpenAI API key pattern in prompt" >&2
  FOUND=1
fi

# AWS access keys
if echo "$PROMPT" | grep -qE 'AKIA[0-9A-Z]{16}'; then
  echo "WARNING: Detected AWS access key pattern in prompt" >&2
  FOUND=1
fi

# Generic high-entropy strings near keywords like "token", "secret", "password"
if echo "$PROMPT" | grep -qiE '(token|secret|password|api_key)\s*[:=]\s*["\x27]?[A-Za-z0-9+/]{20,}'; then
  echo "WARNING: Detected potential secret near sensitive keyword" >&2
  FOUND=1
fi

# GitHub personal access tokens
if echo "$PROMPT" | grep -qE 'ghp_[A-Za-z0-9]{36}'; then
  echo "WARNING: Detected GitHub personal access token in prompt" >&2
  FOUND=1
fi

# Slack tokens
if echo "$PROMPT" | grep -qE 'xox[bpors]-[A-Za-z0-9-]+'; then
  echo "WARNING: Detected Slack token in prompt" >&2
  FOUND=1
fi

if [ "$FOUND" -eq 1 ]; then
  echo "Prompt blocked: contains what appears to be a secret. Remove the secret and try again." >&2
  exit 2
fi

exit 0
```

**Python version (more thorough regex):**

```python
#!/usr/bin/env python3
"""
~/.codex/hooks/redact-secrets.py
Scans the user prompt for secret patterns and blocks if found.
"""

import json
import re
import sys

payload = json.load(sys.stdin)
prompt = payload.get("prompt", "")

SECRET_PATTERNS = [
    # OpenAI keys
    (r'\bsk-[A-Za-z0-9]{20,}\b', "OpenAI API key"),
    # OpenAI project keys
    (r'\bsk-proj-[A-Za-z0-9]{20,}\b', "OpenAI project key"),
    # AWS access keys
    (r'\bAKIA[0-9A-Z]{16}\b', "AWS access key"),
    # AWS secret keys (40 chars, base64-ish)
    (r'(?i)aws_secret_access_key\s*[:=]\s*["\']?([A-Za-z0-9/+=]{40})', "AWS secret key"),
    # GitHub PAT
    (r'\bghp_[A-Za-z0-9]{36}\b', "GitHub personal access token"),
    # GitHub fine-grained PAT
    (r'\bgithub_pat_[A-Za-z0-9_]{22,}\b', "GitHub fine-grained PAT"),
    # Slack tokens
    (r'\bxox[bpors]-[A-Za-z0-9-]+\b', "Slack token"),
    # Generic bearer tokens
    (r'(?i)bearer\s+[A-Za-z0-9._~+/=-]{20,}', "Bearer token"),
    # Private keys
    (r'-----BEGIN (RSA |EC |DSA )?PRIVATE KEY-----', "Private key block"),
    # Generic secrets near keywords
    (r'(?i)(api_key|apikey|secret_key|auth_token|access_token)\s*[:=]\s*["\']?[A-Za-z0-9+/]{16,}',
     "Secret near keyword"),
]

found = []
for pattern, label in SECRET_PATTERNS:
    if re.search(pattern, prompt):
        found.append(label)

if found:
    details = ", ".join(found)
    print(f"Blocked: detected potential secrets in prompt ({details}). "
          f"Remove secrets before submitting.", file=sys.stderr)
    sys.exit(2)

# No secrets found — allow the prompt
sys.exit(0)
```

### Practical Example: Audit All Prompts

For compliance-sensitive environments, log every prompt:

```bash
#!/usr/bin/env bash
# ~/.codex/hooks/audit-prompts.sh
# Logs all user prompts to a JSONL audit file.

INPUT=$(cat)
SESSION_ID=$(echo "$INPUT" | jq -r '.session_id // "unknown"')
TURN_ID=$(echo "$INPUT" | jq -r '.turn_id // "unknown"')
PROMPT=$(echo "$INPUT" | jq -r '.prompt // ""')

LOG_DIR="$HOME/.codex/audit-logs"
mkdir -p "$LOG_DIR"

# Truncate prompt for the log (full prompt may be very large)
PROMPT_PREVIEW=$(echo "$PROMPT" | head -c 500)

echo "$INPUT" | jq --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --arg user "$(whoami)" \
  '. + {logged_at: $ts, user: $user, prompt_length: (.prompt | length)}' \
  >> "$LOG_DIR/prompts-${SESSION_ID}.jsonl"

exit 0
```

---

## Layering Hooks: Global and Repo-Level

Codex loads **all** matching `hooks.json` files from both locations:

- `~/.codex/hooks.json` — user-level hooks that apply to every session
- `<repo>/.codex/hooks.json` — repo-specific hooks

Higher-precedence layers do **not** replace lower-precedence hooks. Both run concurrently.

### Recommended Layering Strategy

**Global (`~/.codex/hooks.json`):** Hooks that should apply everywhere, regardless of project.

- Secret redaction on `UserPromptSubmit`
- Desktop notifications on `Stop`
- Audit logging on all events
- Generic security gates

**Repo-level (`<repo>/.codex/hooks.json`):** Hooks specific to a project's workflow.

- Branch/ticket context injection on `SessionStart`
- Project-specific formatting or linting
- CI integration hooks
- Team-specific content policies

```
~/.codex/
  config.toml          # [features] codex_hooks = true
  hooks.json           # Global: notifications, audit, secret scanning
  hooks/
    redact-secrets.py
    notify-stop.sh
    audit-prompts.sh

~/my-project/.codex/
  hooks.json           # Repo: context injection, project-specific gates
  hooks/
    inject-context.sh
    project-guard.sh
```

This way, you get baseline security and observability everywhere, with project-specific customizations layered on top.

---

## Exit Code Semantics

Exit codes are the primary mechanism hooks use to communicate decisions back to Codex:

| Exit Code | Meaning | Effect |
|-----------|---------|--------|
| **0** | Success / Allow | Hook ran successfully. Any stdout JSON is processed. Any stderr is shown as an informational message. |
| **2** | Block | The triggering action is blocked. Stderr is read as the block reason and displayed to the user. For `UserPromptSubmit`, the prompt is rejected. For `Stop`, this can trigger continuation logic. |
| **Any other non-zero** | Error (non-blocking) | The hook failed but the action proceeds. Stderr is shown as a warning. |

The critical distinction: **exit 1 does not block**. This is a common mistake. If you write a security hook that exits with code 1 when it detects a problem, the problem is logged as a warning but the action still proceeds. Always use exit code 2 for blocking.

### Stdout JSON Responses

Hooks can return structured JSON on stdout. The format depends on the event:

**SessionStart — inject context:**
```json
{
  "additionalContext": "Branch: feature/PROJ-123\nTicket: Implement new auth flow"
}
```

**UserPromptSubmit — modified prompt (future):**
```json
{
  "prompt": "the sanitised version of the prompt"
}
```

**Stop — continuation signal:**
```json
{
  "continue": true,
  "stopReason": "Output was truncated"
}
```

> Note: Not all response fields are implemented for all events in v0.116.0. The `continue` field for `Stop` and prompt modification for `UserPromptSubmit` may have partial support. Test with your specific version.

---

## Practical Example: Full hooks.json for a Team

Here is a complete `hooks.json` that combines all three session-scoped events into a practical team configuration:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": ".codex/hooks/inject-context.sh",
            "timeout": 5,
            "statusMessage": "Loading branch and ticket context..."
          }
        ]
      },
      {
        "matcher": "resume",
        "hooks": [
          {
            "type": "command",
            "command": ".codex/hooks/inject-context.sh",
            "timeout": 5,
            "statusMessage": "Refreshing context for resumed session..."
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.codex/hooks/redact-secrets.py",
            "timeout": 3,
            "statusMessage": "Scanning prompt for secrets..."
          },
          {
            "type": "command",
            "command": "~/.codex/hooks/audit-prompts.sh",
            "timeout": 2
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.codex/hooks/notify-stop.sh",
            "timeout": 5,
            "statusMessage": "Sending notification..."
          }
        ]
      }
    ]
  }
}
```

Both hooks in the `UserPromptSubmit` entry run concurrently: the secret scanner and the audit logger fire at the same time. If the secret scanner exits with code 2 (block), the prompt is rejected regardless of what the audit logger returns.

---

## Debugging Hooks

Hooks run as child processes of the Codex CLI. When things go wrong, here is how to debug:

### 1. Check that hooks are enabled

Verify your `config.toml` has the feature flag:

```bash
cat ~/.codex/config.toml | grep codex_hooks
# Should output: codex_hooks = true
```

### 2. Test hooks manually

Pipe a sample JSON payload to your hook script and check the exit code:

```bash
# Test a SessionStart hook
echo '{"session_id":"test","hook_event_name":"SessionStart","source":"startup","cwd":"/tmp","model":"gpt-5.4"}' \
  | bash ~/.codex/hooks/inject-context.sh
echo "Exit code: $?"

# Test a UserPromptSubmit hook with a secret
echo '{"session_id":"test","hook_event_name":"UserPromptSubmit","prompt":"Use key sk-proj-abc123def456ghi789","turn_id":"test","cwd":"/tmp","model":"gpt-5.4"}' \
  | python3 ~/.codex/hooks/redact-secrets.py
echo "Exit code: $?"
```

### 3. Add logging to your hooks

Write diagnostic output to a log file (not stderr, which Codex displays to the user):

```bash
#!/usr/bin/env bash
INPUT=$(cat)
echo "[$(date)] Hook fired: $(echo "$INPUT" | jq -c '.')" >> /tmp/codex-hook-debug.log

# ... rest of hook logic ...
```

### 4. Watch for timeout kills

If your hook takes longer than the configured `timeout`, Codex kills the process. You will see a warning in the TUI. If your hooks are being killed:

- Reduce the work done in the hook
- Increase the `timeout` value
- Move expensive operations to `Stop` (which is less latency-sensitive)

### 5. Validate your hooks.json

Malformed JSON silently disables hooks. Validate with `jq`:

```bash
jq . ~/.codex/hooks.json >/dev/null && echo "Valid" || echo "Invalid JSON"
jq . .codex/hooks.json >/dev/null && echo "Valid" || echo "Invalid JSON"
```

---

## Limitations

As of v0.116.0, the session-scoped hooks have the following limitations:

1. **No tool-level events.** `SessionStart`, `Stop`, and `UserPromptSubmit` operate at session and turn boundaries. For tool-level interception, you need `PreToolUse` and `PostToolUse`, which shipped in v0.117.0.

2. **No async hooks.** All hooks run synchronously and block the event they are attached to. A slow `UserPromptSubmit` hook delays prompt processing. Async hooks are a planned feature (see [community discussion](https://github.com/openai/codex/discussions/2150)).

3. **No Windows support.** Hooks are disabled on the experimental Windows build.

4. **Concurrent execution only.** Multiple matching hooks run concurrently. You cannot define execution order or make one hook conditional on another's result.

5. **No prompt modification on UserPromptSubmit.** While the protocol includes a `prompt` field in the response JSON, prompt rewriting is not fully supported in v0.116.0. Hooks can block prompts (exit 2) but cannot reliably modify them in-flight.

6. **Stdout must be valid JSON.** If your hook writes non-JSON to stdout, Codex logs a warning and ignores the output. Diagnostic messages should go to stderr or a log file.

7. **Hook scripts must be executable.** Codex spawns hooks via `exec`. If the script lacks execute permissions (`chmod +x`), the hook fails silently.

---

## Summary of the Three Session-Scoped Events

| Event | When It Fires | Matcher | Can Block? | Primary Use Cases |
|-------|--------------|---------|------------|-------------------|
| `SessionStart` | Session opens or resumes | `source` (`startup` / `resume`) | No | Context injection, audit logging, environment setup |
| `UserPromptSubmit` | User submits a prompt | None (fires for all) | Yes (exit 2) | Secret redaction, content policy, prompt auditing |
| `Stop` | Agent turn completes | None (fires for all) | Yes (triggers continuation) | Desktop notifications, Slack alerts, auto-continue |

Together, these three events give you control over the full lifecycle of a Codex session at the session and turn level. Combined with the tool-level hooks (`PreToolUse`/`PostToolUse`) covered in the [companion article](/2026/03/31/codex-cli-pretooluse-posttooluse-hooks/), you have five hook points that cover context injection, security enforcement, quality gates, observability, and notification.

---

## References

- [Codex CLI Hooks Documentation](https://developers.openai.com/codex/hooks) — official reference for all hook events
- [Codex CLI v0.116.0 Release Notes](https://developers.openai.com/codex/changelog) — release that shipped SessionStart, Stop, and UserPromptSubmit
- [PR #15118: Add turn_id to hook payloads](https://github.com/openai/codex/pull/15118) — added turn correlation to UserPromptSubmit and Stop
- [Community Discussion: Async Hooks and Expanded Events](https://github.com/openai/codex/discussions/2150) — feature requests and roadmap discussion
- [Codex CLI PreToolUse & PostToolUse Hooks](/2026/03/31/codex-cli-pretooluse-posttooluse-hooks/) — companion article covering tool-level hooks (v0.117.0)
