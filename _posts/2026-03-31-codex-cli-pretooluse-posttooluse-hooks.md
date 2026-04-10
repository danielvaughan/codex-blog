---
title: "Codex CLI PreToolUse & PostToolUse Hooks: Production Patterns for Quality Gates and CI Enforcement"
date: 2026-03-31
tags:
  - ci-cd
  - hooks
  - testing
  - automation
  - pretooluse
  - posttooluse
  - quality-gates
  - security
---

![Sketchnote diagram for: Codex CLI PreToolUse & PostToolUse Hooks: Production Patterns for Quality Gates and CI Enforcement](/sketchnotes/articles/2026-03-31-codex-cli-pretooluse-posttooluse-hooks.png)

# Codex CLI PreToolUse & PostToolUse Hooks: Production Patterns for Quality Gates and CI Enforcement


---

The Codex CLI hooks engine gained two new events in v0.117.0: `PreToolUse` and `PostToolUse`.[^1] Unlike `SessionStart`, `Stop`, and `userpromptsubmit` (which were the first three hooks shipped), these two events fire at the tool level — wrapping every Bash command the agent runs. They are the hook events that enable production-grade quality gates: blocking destructive commands before they run, enforcing formatting after every file write, and capturing tool-level telemetry for audit trails.

This article documents the configuration format, blocking semantics, and four practical patterns that cover the most common production requirements.

---

## What Changed

The [earlier hooks deep-dive](./2026-03-26-codex-cli-hooks-deep-dive.md) covers `SessionStart`, `Stop`, and `userpromptsubmit` — events that fire at session or turn boundaries. PreToolUse and PostToolUse fire at a finer grain: every time the model calls a tool.

| Hook Event | When | Can Block? | Use for |
|-----------|------|-----------|---------|
| `SessionStart` | Session opens or resumes | No | Context injection, audit log entry |
| `userpromptsubmit` | User submits a prompt | Yes | Prompt sanitisation, content policy |
| `PreToolUse` | Before tool executes | Yes (exit 2) | Security gates, dangerous command blocks |
| `PostToolUse` | After tool completes | No (undo impossible) | Formatting, quality validation, telemetry |
| `Stop` | Turn ends | Yes (triggers continuation) | Completeness checks, auto-follow-up |

---

## Configuration

Hooks live in `.codex/hooks.json` (repo-level) or `~/.codex/hooks.json` (user-level). Both files are merged; all matching hooks for an event run concurrently. The feature flag must be enabled:

```toml
# .codex/config.toml
[features]
codex_hooks = true
```

The `matcher` field is a regex applied to the tool name (and for Bash, to the command string). The currently supported hooks for matchers are `PreToolUse`, `PostToolUse`, and `SessionStart`.[^2]

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.codex/hooks/pre-bash-guard.sh",
            "timeout": 10
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.codex/hooks/post-bash-telemetry.sh",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

---

## Blocking Semantics: Exit Code 2, Not Exit Code 1

This is the most common mistake when writing PreToolUse hooks.

- **Exit 0** — allow; hook feedback (if any) is surfaced as a warning but execution continues
- **Exit 2** — **block**: Codex stops the tool call, reads your `stderr` as the block reason, and feeds that reason to the model as context so it can try a different approach
- **Any other non-zero** — non-blocking error; shown to the user as a warning but the tool still runs

If you write a security gate with `exit 1` instead of `exit 2`, the gate appears to work (your warning prints to the terminal) — but the dangerous command still executes. Always use `exit 2` for blocks.[^3]

```bash
#!/usr/bin/env bash
# ~/.codex/hooks/pre-bash-guard.sh
# Blocks destructive patterns before Codex runs them.

INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // ""')

# Block hard deletes
if echo "$COMMAND" | grep -qE 'rm\s+-rf\s+/|rm\s+-rf\s+~'; then
  echo "BLOCKED: Recursive root/home delete detected" >&2
  exit 2
fi

# Block force pushes to main/master
if echo "$COMMAND" | grep -qE 'git\s+push\s+.*--force.*\s+(main|master)'; then
  echo "BLOCKED: Force push to protected branch" >&2
  exit 2
fi

exit 0
```

---

## Pattern 1 — Security Gate (PreToolUse)

Block a curated list of high-risk commands. The hook receives a JSON object on stdin with `tool_input.command` set to the Bash command the model wants to run.

```bash
#!/usr/bin/env bash
# Deny-list approach: known-dangerous patterns only.
# Complements (does not replace) Codex's sandbox approval modes.

INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // ""')

PATTERNS=(
  'DROP\s+TABLE'
  'truncate\s+--wipefs'
  'dd\s+if=.*of=/dev/'
  'chmod\s+-R\s+777\s+/'
)

for pat in "${PATTERNS[@]}"; do
  if echo "$CMD" | grep -qiE "$pat"; then
    echo "Security gate: command matches blocked pattern '$pat'" >&2
    exit 2
  fi
done

exit 0
```

Keep this hook fast — it gates every Bash call. Target under 200ms. A slow PreToolUse hook visibly degrades interactive feel.

---

## Pattern 2 — Auto-Format After Writes (PostToolUse)

Run your formatter after every file write. PostToolUse fires after the tool completes, so it cannot undo side effects — but it can trigger follow-up actions. Since formatting is idempotent, running it unconditionally is safe.

```bash
#!/usr/bin/env bash
# post-format.sh — run prettier/ruff after Bash writes.

INPUT=$(cat)
EVENT=$(echo "$INPUT" | jq -r '.hook_event_name // ""')
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // ""')

# Only act on write-like commands
if echo "$CMD" | grep -qE '(>|tee|cat\s+>|write_file)'; then
  # Run formatter on changed files
  CHANGED=$(git diff --name-only 2>/dev/null)
  if [ -n "$CHANGED" ]; then
    echo "$CHANGED" | xargs -I{} sh -c '
      case "$1" in
        *.py) ruff format "$1" ;;
        *.ts|*.tsx|*.js|*.jsx) npx prettier --write "$1" ;;
        *.go) gofmt -w "$1" ;;
      esac
    ' -- {}
  fi
fi

exit 0
```

---

## Pattern 3 — Audit Log (PostToolUse)

Write a JSONL audit trail of every tool call the agent makes. Useful for compliance, debugging, and understanding what the agent actually did in a long session.

```bash
#!/usr/bin/env bash
# post-audit.sh — append tool calls to a session audit log.

INPUT=$(cat)
LOG_DIR="${CODEX_AUDIT_LOG_DIR:-$HOME/.codex/audit-logs}"
mkdir -p "$LOG_DIR"

SESSION_ID=$(echo "$INPUT" | jq -r '.session_id // "unknown"')
LOG_FILE="$LOG_DIR/${SESSION_ID}.jsonl"

# Append enriched record
echo "$INPUT" | jq --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  '. + {logged_at: $ts}' >> "$LOG_FILE"

exit 0
```

The audit log accumulates one JSON record per tool call. You can post-process it with `jq` to extract patterns, measure tool call frequency, or replay a session for debugging.

---

## Pattern 4 — Quality Gate at Git Operations (PreToolUse)

Run tests before the agent commits. Block the commit if tests fail, letting the model see the failure output and attempt a fix.

```bash
#!/usr/bin/env bash
# pre-commit-guard.sh — block git commits if tests fail.

INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // ""')

# Only intercept commit commands
if ! echo "$CMD" | grep -qE '^git\s+commit'; then
  exit 0
fi

# Run the test suite
if command -v pytest >/dev/null 2>&1; then
  OUTPUT=$(pytest --tb=short -q 2>&1)
  if [ $? -ne 0 ]; then
    echo "Tests failed — blocking commit:" >&2
    echo "$OUTPUT" >&2
    exit 2
  fi
elif command -v npm >/dev/null 2>&1 && [ -f package.json ]; then
  OUTPUT=$(npm test --silent 2>&1)
  if [ $? -ne 0 ]; then
    echo "Tests failed — blocking commit:" >&2
    echo "$OUTPUT" >&2
    exit 2
  fi
fi

exit 0
```

When the model tries to commit and tests fail, it receives the test output as context and will typically attempt to fix the failures before retrying.

---

## Subagent Enforcement

Hooks fire for subagent tool calls too.[^3] When Codex spawns a subagent (via the multi-agent v2 path-addressing system), your PreToolUse and PostToolUse hooks execute for every tool the subagent calls. This is critical for security gates — without it, a subagent could bypass your policy by being spawned outside the hook scope.

No extra configuration is needed; the hooks apply to the full session tree.

---

## Performance Guidance

Since PreToolUse runs synchronously on every matched tool call, latency matters:

- **Under 200ms** — imperceptible; safe to run on every call
- **200–500ms** — noticeable on interactive use; consider `matcher` to narrow scope
- **Over 500ms** — use PostToolUse instead (async-friendly) or tighten the matcher

For intensive checks (full test suite, lint across many files), use PostToolUse or the `Stop` hook, where latency doesn't gate interactive response.

---

## Limitations (March 2026)

- `PostToolUse` currently **supports Bash tool results only** (not file-write tools directly)[^2]
- Hooks are **disabled on Windows**
- Multiple matching hooks run concurrently — one cannot prevent another from starting
- PostToolUse **cannot undo** a completed command; for blocking, use PreToolUse

---

## See Also

- [Codex CLI Hooks Deep Dive: SessionStart, Stop and userpromptsubmit](./2026-03-26-codex-cli-hooks-deep-dive.md) — the companion article covering session-scoped hooks
- Official hooks reference: [developers.openai.com/codex/hooks](https://developers.openai.com/codex/hooks)

---

[^1]: Codex CLI v0.117.0 changelog, March 2026. "Non-streaming (non-stdin style) shell-only PostToolUse support" added. PreToolUse available from v0.114.0. [developers.openai.com/codex/changelog](https://developers.openai.com/codex/changelog)
[^2]: Official Codex hooks documentation. [developers.openai.com/codex/hooks](https://developers.openai.com/codex/hooks). Retrieved 2026-03-31.
[^3]: Codex CLI GitHub issue #14754: "Add PreToolUse and PostToolUse hook events for code quality enforcement". [github.com/openai/codex/issues/14754](https://github.com/openai/codex/issues/14754)
