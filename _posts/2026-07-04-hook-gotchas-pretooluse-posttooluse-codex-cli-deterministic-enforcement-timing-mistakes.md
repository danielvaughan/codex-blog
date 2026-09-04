---
title: "Hook Gotchas: Seven PreToolUse and PostToolUse Timing Mistakes That Break Deterministic Enforcement in Codex CLI"
description: "Hooks are the deterministic enforcement layer that makes agent compliance guaranteed rather than probable. But misconfiguring hook timing, scope, and exit codes silently downgrades your guardrails from ironclad to advisory."
type: Technical Article
timestamp: 2026-07-04T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-07-04-hook-gotchas-pretooluse-posttooluse-codex-cli-deterministic-enforcement-timing-mistakes"
tags: ["codex-cli", "hooks", "PreToolUse", "PostToolUse", "deterministic-enforcement", "anti-patterns", "security", "guardrails"]
date: 2026-07-04T09:00:00+00:00
last_modified_at: 2026-09-04T10:27:33+01:00
---
# Hook Gotchas: Seven PreToolUse and PostToolUse Timing Mistakes That Break Deterministic Enforcement in Codex CLI


---

Hooks are the deterministic enforcement layer that makes agent compliance guaranteed rather than probable. When a prompt says "never write secrets to files", there is always a non-zero probability the model ignores it. When a PreToolUse hook exits with code 2, the tool call is blocked with mathematical certainty. No probability. No retries. No reasoning.

This makes hooks the single most powerful compliance mechanism in Codex CLI. It also makes misconfiguring them uniquely dangerous — because a broken hook does not fail loudly. It fails silently, downgrading your "guaranteed block" to "nothing happened at all."

After reviewing community issue reports, enterprise deployment audits, and the patterns documented in the Claude Certified Architect exam preparation materials, seven hook mistakes recur consistently. Each one either (a) silently disables enforcement, (b) blocks the wrong things, or (c) creates performance problems that lead teams to disable hooks entirely.

---

## Gotcha 1: Using PostToolUse When You Need PreToolUse

**The mistake:** Putting compliance logic in a PostToolUse hook to prevent dangerous operations.

**Why it breaks:** PostToolUse fires *after* the tool has already executed. If your hook checks whether a file write contains secrets, the file has already been written by the time the hook runs. Your "block" is actually a "detect after the damage is done."

**The correct pattern:**

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash .hooks/pre-write-secrets-check.sh",
            "timeout": 5000
          }
        ]
      }
    ]
  }
}
```

PreToolUse receives the tool call *before* execution. Exit code 2 blocks the call entirely. The tool never runs. The file is never written. The secret never touches the filesystem.

**The CCA-F principle:** "When the question uses words like must, never, or guarantee, the guarantee always wins." If something must never happen, PreToolUse is the only option. PostToolUse can only report, transform, or clean up — it cannot prevent.

---

## Gotcha 2: Assuming Exit Code 1 Blocks Execution

**The mistake:** Writing a PreToolUse hook that exits with code 1 when it detects a violation, expecting the tool call to be blocked.

**Why it breaks:** In Codex CLI's hook model, exit codes have specific meanings:

| Exit code | Effect in PreToolUse | Effect in PostToolUse |
|-----------|---------------------|----------------------|
| 0 | Allow (proceed) | Accept output as-is |
| 1 | Hook error (logged, but tool still runs) | Hook error (logged, output unchanged) |
| 2 | **Block** (tool call prevented) | Transform/reject output |

Exit code 1 means "the hook itself failed" — a script error, missing dependency, or timeout. Codex CLI treats this as a hook malfunction, not as a deliberate enforcement decision. The tool call proceeds on the assumption that you would rather have a working agent with a broken hook than a completely stuck agent.

**The fix:** Always use `exit 2` for deliberate blocks:

```bash
#!/bin/bash
# .hooks/pre-write-secrets-check.sh

# Read the proposed file content from stdin
CONTENT=$(cat)

# Check for secret patterns
if echo "$CONTENT" | grep -qE '(AKIA[A-Z0-9]{16}|ghp_[a-zA-Z0-9]{36}|sk-[a-zA-Z0-9]{48})'; then
  echo "BLOCKED: Detected potential secret in proposed file write" >&2
  exit 2  # EXIT 2 = deterministic block
fi

exit 0  # EXIT 0 = allow
```

---

## Gotcha 3: Setting Timeouts Too Short for Meaningful Checks

**The mistake:** Using the default or a very short timeout (1000-2000ms) for hooks that run Gradle, Docker, or any compilation step.

**Why it breaks:** When a hook times out, it is treated as exit code 1 (hook error) — which means the tool call proceeds. Your 30-second ArchUnit check that verifies layer boundaries never finishes, and the enforcement silently disappears.

Teams often discover this only when reviewing logs weeks later: "Why did the hook allow a domain-to-adapter dependency? Oh — it timed out on every invocation and was never actually checking anything."

**The fix:** Set timeouts proportional to what the hook actually does:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash .hooks/post-write-check.sh",
            "timeout": 120000
          }
        ]
      }
    ]
  }
}
```

| Check type | Realistic timeout |
|-----------|------------------|
| Secret scanning (regex) | 5,000ms |
| Lint/format check | 15,000ms |
| Unit test compilation | 30,000ms |
| Full test suite | 120,000ms |
| Container build + scan | 300,000ms |

**Rule of thumb:** Measure your hook's p99 execution time, then set the timeout to 3x that value. If your check cannot run within 5 minutes, it belongs in a `./gradlew verify` task triggered less frequently, not on every file write.

---

## Gotcha 4: Running Expensive Hooks on Every Tool Call

**The mistake:** Attaching a full `./gradlew test` run to PostToolUse without filtering which tool calls trigger it.

**Why it breaks:** PostToolUse fires on *every* tool call — reads, writes, bash commands, search operations. If your hook runs the full test suite every time the agent reads a file, a session that would take 5 minutes takes 3 hours. The team disables hooks "temporarily" to unblock the agent. Temporarily becomes permanently.

**The fix:** Filter hooks by tool name or by what changed:

```bash
#!/bin/bash
# .hooks/post-write-check.sh
# Only run checks when files were actually modified

TOOL_NAME="${CODEX_HOOK_TOOL_NAME:-}"

# Skip non-write operations
case "$TOOL_NAME" in
  write|edit|apply_patch) ;;
  *) exit 0 ;;
esac

# Only check modified source files, not config or docs
CHANGED_FILE="${CODEX_HOOK_FILE_PATH:-}"
case "$CHANGED_FILE" in
  src/main/java/*|src/test/java/*) ;;
  *) exit 0 ;;
esac

# Now run the targeted check
./gradlew spotlessCheck checkstyleMain -q 2>&1 || exit 2
```

**The principle:** Hooks should be fast-path by default. Only do expensive work when the trigger justifies it.

---

## Gotcha 5: Using Prompt Hooks When You Need Command Hooks

**The mistake:** Using hook types that involve LLM evaluation ("prompt" or "agent" hooks) for compliance enforcement.

**Why it breaks:** The CCA-F exam materials make this distinction explicit and testable:

- **Command hooks** (`"type": "command"`) are deterministic. They run a shell script. The exit code decides. No LLM involved. 100% reliable.
- **Prompt hooks** are probabilistic. They ask the model "should this be allowed?" The model's answer has a non-zero failure rate.

If your compliance requirement is "never write AWS credentials to files," a prompt hook might evaluate to "this looks like test data, allow it" on a well-crafted test fixture that happens to contain real credentials. A command hook running `gitleaks` will catch it every time regardless of what the model thinks about context.

**The rule:** For anything that uses the words "must", "never", "always", or "guaranteed" — use command hooks only. Reserve prompt/agent hooks for advisory guidance ("suggest better variable names"), not enforcement.

---

## Gotcha 6: Not Passing Context to the Hook Script

**The mistake:** Writing hook scripts that check for violations but have no information about *what* is being written or *where*.

**Why it breaks:** A PreToolUse hook receives information about the proposed tool call via stdin or environment variables. If your script ignores this context and instead scans the whole repository, it is (a) slow, (b) checking unchanged files, and (c) missing the actual content of the proposed write.

**The fix:** Use the hook's input context:

```bash
#!/bin/bash
# PreToolUse hook receives the tool call payload on stdin
PAYLOAD=$(cat)

# Extract the file path and content from the JSON payload
FILE_PATH=$(echo "$PAYLOAD" | jq -r '.file_path // empty')
CONTENT=$(echo "$PAYLOAD" | jq -r '.content // empty')

# Target check: only scan the specific content being written
if [ -n "$CONTENT" ]; then
  echo "$CONTENT" | gitleaks detect --pipe --no-banner 2>/dev/null
  if [ $? -ne 0 ]; then
    echo "BLOCKED: Secret detected in content being written to $FILE_PATH" >&2
    exit 2
  fi
fi

exit 0
```

---

## Gotcha 7: Assuming Hooks Work in the API

**The mistake:** Writing hook logic assuming it will intercept tool calls when using the Claude/OpenAI API directly (not through Claude Code or Codex CLI).

**Why it breaks:** Hooks are a feature of the *client tool* (Codex CLI or Claude Code), not the underlying API. The API has no interception layer — tool calls are returned in the response, and your application code decides what to do with them. There is no PreToolUse event at the API level.

If you are building a custom agent harness on the raw API, you need to implement your own tool call validation in your application code — typically as a middleware layer between the API response parser and the tool executor:

```python
# Custom harness equivalent of PreToolUse
def execute_tool_call(tool_call):
    # Your enforcement logic here (equivalent to hook scripts)
    if tool_call.name == "write_file":
        if contains_secrets(tool_call.arguments["content"]):
            return {"is_error": True, "content": "BLOCKED: secret detected"}

    # Proceed with execution
    return actually_execute(tool_call)
```

**The principle:** Hooks are a Codex CLI / Claude Code convenience layer over a pattern you would otherwise implement manually. If you move to a custom harness, you must rebuild the enforcement — it does not come free from the API.

---

## The Meta-Pattern: Enforcement vs Advisory

Every hook configuration decision reduces to one question: **Is this a guarantee or a suggestion?**

| Requirement | Enforcement type | Hook approach |
|------------|-----------------|---------------|
| "Must never contain secrets" | Guarantee | PreToolUse, command, exit 2 |
| "Should follow naming conventions" | Advisory (strong) | PostToolUse, command, warning in output |
| "Might want to consider shorter methods" | Advisory (soft) | No hook — put it in AGENTS.md |
| "Must pass all tests before commit" | Guarantee | PostToolUse on commit-like operations, command, exit 2 |
| "Should generate documentation" | Advisory | Prompt in AGENTS.md |

If you blur this boundary — using prompt-based enforcement for guarantees, or using deterministic hooks for soft suggestions — you either get false security (thinking something is blocked when it is not) or frustrated agents (blocked from reasonable actions by overly rigid gates).

The hooks are the closest thing to a type system for agent behaviour. Use them like one: strict where it matters, absent where it doesn't.

---

## References

- OpenAI, "Hooks — Codex CLI," developers.openai.com, 2026. PreToolUse/PostToolUse event model, exit code semantics, timeout handling.
- Anthropic, "Claude Code Hooks," code.claude.com, 2026. The same hook model in Claude Code — confirms the shared enforcement pattern.
- CCA-F exam anti-patterns (Domain 1, Domain 3). "Using prompts for guaranteed compliance" is a confirmed distractor; the correct answer is always deterministic enforcement via hooks.
