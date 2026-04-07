---
title: "Debugging Codex Agent Failures: A Systematic Troubleshooting Guide"
date: 2026-03-28
tags: [debugging, troubleshooting, api-errors, context-rot, compaction, agent-failures]
---

# Debugging Codex Agent Failures: A Systematic Troubleshooting Guide

Codex CLI agent failures cluster into a small number of recognisable patterns. Most failures are not random — they have consistent causes and systematic fixes. This guide covers the most common failure modes observed across community reports, with decision trees and preventive configuration patterns for each.

---

## Failure Mode 1: API Errors (18% of reported bugs)

**Symptoms:** Agent stops mid-task with `Error: 429 Too Many Requests`, `503 Service Unavailable`, or `500 Internal Server Error`. Progress is lost.

**Root causes:**
- Rate limit exceeded (most common for heavy subagent workloads)
- Transient API availability issue
- Quota exhausted for the session

**Diagnosis:**

```bash
# Check your current model and usage mode
codex config get model
codex config get model_reasoning_effort

# Run with verbose output to see raw API responses
CODEX_LOG_LEVEL=debug codex "your task" 2>&1 | grep -i "error\|rate\|429\|503"
```

**Fixes:**

1. **Rate limiting:** Reduce parallelism. Set `max_threads = 2` in your `[subagents]` config block. Route worker agents to `gpt-5-codex-mini` or `gpt-5.4-mini` for lower rate consumption.

2. **Quota exhausted:** Use `codex resume --last` to continue from where the session stopped. The `--last` flag resumes the most recent thread without repeating completed work.

3. **Transient failures:** Add a retry hook. In `~/.codex/config.toml`:

```toml
[hooks.SessionStart]
command = "echo 'Session starting' >> /tmp/codex-sessions.log"

[hooks.Stop]
command = "echo 'Session stopped at $(date)' >> /tmp/codex-sessions.log"
```

4. **For CI/CD:** Add `--attempts 3` to `codex cloud exec` for best-of-N retry semantics.

**Preventive pattern:** Set `max_tokens_per_session` in config to get early warning before quota exhaustion causes a hard stop mid-task.

---

## Failure Mode 2: Terminal and PTY Problems (14% of reported bugs)

**Symptoms:** Commands hang indefinitely, output is garbled, interactive commands stall, `apply_patch` fails unexpectedly.

**Root causes:**
- PTY (pseudo-terminal) allocation issues
- Commands that expect interactive input
- `apply_patch` regression in v0.117.0 (Issue #16102)

**Diagnosis:**

```bash
# Check if unified_exec is enabled (PTY-backed exec)
codex features list | grep unified_exec

# Test if the issue is patch-specific
codex config get features.unified_exec
```

**Fixes:**

1. **`apply_patch` failures (v0.117.0):** Known regression. Workaround: instruct Codex to use direct file writes instead of patch operations by adding to your `AGENTS.md`:
   ```
   Prefer writing complete file contents over patch operations when making changes.
   ```

2. **Hanging commands:** Ensure `unified_exec` is enabled (`features.unified_exec = true`). This gives Codex a proper PTY environment for command execution.

3. **Interactive commands stalling:** Codex is not designed to handle interactive commands (those that wait for keyboard input). Add to your `AGENTS.md`:
   ```
   Do not run interactive commands. Use non-interactive flags (e.g., `npm install --yes`, `git commit -m "message"` not `git commit`).
   ```

4. **Windows-specific:** The PowerShell parser process reuse fix merged in v0.118.0 resolves performance issues on Windows. Update when v0.118.0 stable ships.

---

## Failure Mode 3: Command Execution Failures (13% of reported bugs)

**Symptoms:** Agent reports `command not found`, wrong working directory, environment variables missing, build tools behave differently than expected.

**Root causes:**
- Shell environment not matching developer's environment
- Working directory assumptions
- Missing tools not in the sandbox PATH

**Diagnosis:**

```bash
# Check what environment Codex sees
codex "run 'echo $PATH && which node && pwd' and show me the output"
```

**Fixes:**

1. **PATH issues:** Add your tool paths explicitly to your `AGENTS.md` or set them in a `SessionStart` hook:
   ```toml
   [hooks.SessionStart]
   command = "export PATH=$HOME/.nvm/versions/node/v20.0.0/bin:$PATH"
   ```

2. **Working directory:** Always use `codex --cd /path/to/project "task"` in non-interactive mode, or set `default_cwd` in your config.

3. **Missing tools in sandbox:** The Codex sandbox restricts which executables can run. Add required tools to your sandbox allowlist:
   ```toml
   [sandbox]
   allowed_executables = ["node", "npm", "python3", "pytest", "cargo"]
   ```

---

## Failure Mode 4: Auto-Compaction Stalls

**Symptoms:** Agent suddenly becomes slow or stops making progress. Context window is large. Agent begins repeating earlier work. Performance degrades noticeably mid-session.

**Root causes:**
- Context window nearing limit, auto-compaction triggered
- Compaction produces a summary that loses critical technical detail
- The compacted context misses earlier decisions, causing the agent to re-investigate

**Diagnosis:**

Look for compaction events in the TUI (`/session` shows context usage). A compaction that drops from 95% to 30% utilisation mid-task is a signal.

**Fixes:**

1. **Prevent runaway context:** Use `/compact` proactively before the window fills — better to compact on your terms than have auto-compaction at a critical moment.

2. **Preserve key context:** Before `/compact`, ask Codex to write a `PROGRESS.md` or `CONTEXT.md` file:
   ```
   Before we compact, please write a PROGRESS.md file documenting: (1) what we've accomplished, (2) key decisions made, (3) what remains to do, (4) any gotchas discovered.
   ```
   This file survives compaction and gives the agent a readable reference.

3. **Delegate to subagents:** For tasks where context rot is a risk, use subagents for isolated subtasks. Each subagent starts with a fresh context:
   ```toml
   # team.toml
   [[agents]]
   name = "implementer"
   prompt = "Implement the feature described in SPEC.md. Write tests first."
   ```

4. **4-file durable memory pattern:** For very long-horizon tasks (25+ hours), maintain four persistent files: `GOAL.md`, `PROGRESS.md`, `DECISIONS.md`, `BLOCKERS.md`. Have the agent update these regularly. The files persist across compaction and session resume.

---

## Failure Mode 5: Context Rot

**Symptoms:** Agent outputs become repetitive or contradictory. Agent "forgets" earlier constraints. Quality degrades after many turns. Agent proposes changes it already reversed.

**Root causes:**
- Too much accumulated tool-call history filling the context
- Contradictory instructions building up across turns
- The quadratic context growth problem (each new token must attend to all previous tokens)

**Diagnosis:** Context rot typically sets in after 80–150 turns in a complex session.

**Fixes:**

1. **Start fresh for unrelated tasks:** Don't reuse a long session for a new task. `codex` (new session) is better than extending a 200-turn thread.

2. **Fork at key decision points:** Use `/fork` to create a clean branch from the current state. The fork inherits context but starts clean from that point.

3. **Keep AGENTS.md lean:** Counterintuitively, an AGENTS.md that is too long contributes to context rot. Aim for under 500 words. Put detailed conventions in separate `CONVENTIONS.md` files that Codex loads on demand via skills.

4. **Use `/compact` early:** Don't wait for auto-compaction. Compact at natural breakpoints (feature complete, tests passing) to clear the tool-call history while preserving the key results.

---

## Failure Mode 6: Tool Invocation Failures

**Symptoms:** Agent reports that a tool is unavailable, MCP server not responding, or tool call returns unexpected errors.

**Root causes:**
- MCP server not running or misconfigured
- Tool name changed between versions
- Sandbox blocking the tool's network access

**Diagnosis:**

```bash
# List available tools in current session
codex "list all available tools in this session"

# Check MCP server status
codex mcp list
codex mcp test <server-name>
```

**Fixes:**

1. **MCP server not responding:** Restart the MCP server process. Check `~/.codex/config.toml` for the `[mcp_servers]` configuration and ensure the server is running.

2. **Tool not found:** Check for tool naming changes. `read_file` and `grep_files` were retired in v0.116.0–v0.117.0. The subagent tool is now consistently `wait_agent` (not `waitAgent` or `wait-agent`).

3. **Network blocked:** By default, Codex CLI sandbox restricts network access. If a tool needs network access, either configure the sandbox or use `request_permissions` at runtime:
   ```
   /grant network:<hostname>
   ```

---

## Quick Reference: Decision Tree

```
Agent stopped unexpectedly
├── API error (429/503/500) → check rate limits, resume session, reduce parallelism
├── Command hung → check PTY/unified_exec, avoid interactive commands
├── apply_patch failed → v0.117.0 bug, use file writes instead
├── Quality degraded mid-session
│   ├── Context usage >80% → compact proactively
│   ├── Many turns (>100) → fork or start fresh
│   └── Agent repeating work → check for context rot, write PROGRESS.md
└── Tool unavailable
    ├── MCP error → restart MCP server, check config
    ├── Tool renamed → check changelog for API changes
    └── Network blocked → grant sandbox network permission
```

---

## Preventive Configuration

Add these patterns to `~/.codex/config.toml` to reduce failure rates:

```toml
# Limit session size to get early warning
max_tokens_per_session = 100000

# Use PTY-backed execution
[features]
unified_exec = true

# Log session events for post-mortem analysis
[hooks.SessionStart]
command = "echo '{\"event\": \"start\", \"time\": \"'$(date -Iseconds)'\"}' >> /tmp/codex-audit.jsonl"

[hooks.Stop]
command = "echo '{\"event\": \"stop\", \"time\": \"'$(date -Iseconds)'\"}' >> /tmp/codex-audit.jsonl"

# Limit subagent parallelism to avoid rate limits
[subagents]
max_threads = 3
```

---

## v0.117.0 Known Bugs to Work Around

Until v0.118.0 ships, these are the active bugs to be aware of:

| Issue | Symptom | Workaround |
|-------|---------|------------|
| #16102 | `apply_patch` failures | Use direct file writes |
| #16092 | Ghost subagent threads in TUI | Restart session to clear |
| #16093 | Automation skill links stripped | One `@skill` reference per prompt |
| #16091 | Mention picker shows only 8 skills | Type skill name directly |
| #16099 | High GPU usage on macOS | Minimise to tray when idle |

---

*For the latest bug reports and fixes, watch [github.com/openai/codex/issues](https://github.com/openai/codex/issues) and [github.com/openai/codex/milestone/next](https://github.com/openai/codex/milestone/next).*
