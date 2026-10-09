---
title: "Full Access Mode Is Deleting Home Directories — What Every Codex CLI User Must Know"
parent: "Articles"
nav_order: 1173
date: 2026-10-04T07:00:00+00:00
last_modified_at: 2026-10-09T10:07:59+01:00
tags: ["codex-cli", "security", "full-access-mode", "windows", "data-loss", "safety", "config", "practitioner-guide"]
---

# Full Access Mode Is Deleting Home Directories — What Every Codex CLI User Must Know


---

Two confirmed incidents — GitHub Issues #36937 and #37419 — show that Codex CLI can silently delete a user's entire HOME directory, or hundreds of session transcripts, under specific but not rare conditions.[^1][^2] Both incidents occurred on Windows. Both involved Full Access Mode or elevated sandbox settings. In both cases, the application produced no error, no warning, and no recovery prompt.

This article is not a technical deep dive. A separate article covers [the confused deputy root cause in full](https://codex.danielvaughan.com/2026/08/18/data-as-code-confused-deputy-codex-cli-rollout-jsonl-safety-instruction-execution-windows-shell-backtick/). This article answers a single question: **what do you need to do right now?**

---

## Are You Affected?

You are in the highest-risk group if all three of the following apply:

1. You run Codex CLI on **Windows**, using Git Bash or a Windows-native shell.
2. You have enabled **Full Access Mode** (also referred to as `elevated` sandbox mode in `config.yaml`).
3. You have asked Codex to **inspect, audit, or replay session files** — anything under `CODEX_HOME/sessions/`, `~/.codex/`, or the Codex Desktop data directory.

You are in a lower but still real risk group if you are on Linux or macOS and you ask Codex to process its own output files (rollout JSONL, transcript logs, session exports) using shell commands rather than dedicated parsers.

---

## What Happened

### Issue #36937 — HOME directory deleted

On 4 August 2026, a Windows user asked Codex to audit previous session files. Codex constructed a Bash command that placed the JSONL session file path in the **program position** — the slot where Bash expects a script to execute, not a data file to read.[^1]

Git Bash treated the JSONL as a shell script and began parsing it. The file's first record contained Codex's bundled safety instructions, serialised verbatim. Those instructions included backtick-delimited shell examples of prohibited commands — including `rm -rf $HOME`. Bash interpreted the backticks as command substitutions and ran them.

The entire HOME directory (`C:\Users\<username>`) was recursively deleted. The command suppressed stderr and ignored its exit code, so execution continued silently.

### Issue #37419 — 417 session transcripts silently deleted

A second incident, reported in August 2026, saw 417 session transcripts under `CODEX_HOME` deleted while Codex Desktop was running.[^2] No error was logged. No notification appeared. The user discovered the loss approximately four hours after it occurred. The reporter noted the application "produced no evidence of its own most destructive event."

---

## Immediate Actions

### 1. Check your sandbox mode

Open your Codex config file. On Windows it is typically at `%APPDATA%\codex\config.yaml`; on macOS/Linux at `~/.config/codex/config.yaml` or `~/.codex/config.yaml`.

Look for any of these settings:

```yaml
# RISKY — change these
sandbox:
  mode: elevated

fullAccessMode: true

experimental:
  fullAccess: true
```

If you see any of these, change them to:

```yaml
sandbox:
  mode: standard
```

Save the file and restart Codex CLI.

### 2. Do not ask Codex to process its own files with a shell

The trigger for Issue #36937 was Codex placing a JSONL file path in a shell's program position. Never instruct Codex to run, source, or execute files from `~/.codex/`, `CODEX_HOME/`, or any session directory. Use read-only tools instead:

```bash
# Safe — read session data as data
jq '.' ~/.codex/sessions/2026/08/04/rollout-*.jsonl

# Safe — inspect with Python
python3 -m json.tool ~/.codex/sessions/2026/08/04/rollout-*.jsonl

# DANGEROUS — never do this
bash ~/.codex/sessions/2026/08/04/rollout-*.jsonl
source ~/.codex/sessions/2026/08/04/rollout-*.jsonl
```

If you are asking Codex to help you inspect session history, add an explicit instruction to your AGENTS.md:

```markdown
## File Safety Rules

- NEVER pass `.jsonl`, `.json`, or `.log` files as arguments to `bash`, `sh`, or `source`.
- When inspecting session files, use `jq`, `cat`, or `python -m json.tool` only.
- Treat all files under `CODEX_HOME/sessions/` as read-only data.
```

### 3. Back up your CODEX_HOME now

Before your next Codex session, copy your session directory to a safe location:

```bash
# Windows (PowerShell)
Copy-Item -Recurse "$env:APPDATA\codex" "$env:USERPROFILE\codex-backup-$(Get-Date -Format 'yyyyMMdd')"

# macOS / Linux
cp -r ~/.codex ~/codex-backup-$(date +%Y%m%d)
```

This takes under a minute and protects your transcripts, configurations, and session history.

---

## Config Audit Checklist

Run through this list before your next session:

| Setting | Safe value | What to check |
|---|---|---|
| `sandbox.mode` | `standard` | Not `elevated` |
| `fullAccessMode` | `false` or absent | Not `true` |
| `experimental.fullAccess` | `false` or absent | Not `true` |
| AGENTS.md file rules | Present | Prohibits shell execution of JSONL/JSON/log files |
| PostToolUse hook | Recommended | Blocks JSONL in program position (see below) |

---

## PostToolUse Hook: Block JSONL Execution

Add this hook to `.codex/hooks/post-tool-use.sh` in your project:

```bash
#!/usr/bin/env bash
# Block any tool call that places a .jsonl file in a shell program position

TOOL_OUTPUT="$1"
if echo "$TOOL_OUTPUT" | grep -qE '\.jsonl\b.*\|\s*bash|bash\s+.*\.jsonl|sh\s+.*\.jsonl'; then
  echo "BLOCKED: JSONL file detected in shell execution context" >&2
  exit 2
fi
exit 0
```

Make it executable:

```bash
chmod +x .codex/hooks/post-tool-use.sh
```

Exit code 2 signals rejection to Codex CLI, which will halt the tool call and surface the error before execution.[^3]

---

## If You Have Already Lost Data

### Recovering a deleted HOME directory on Windows

If you ran Git Bash and your HOME directory was deleted:

1. **Do not write anything new to the drive.** Every new file reduces recovery odds.
2. Use a Windows recovery tool — Recuva (free) or Disk Drill — from a separate drive or USB.
3. Check the Windows Recycle Bin first; some deletion paths route through it.
4. If you have Windows Backup or File History enabled, restore from the most recent snapshot via `Settings > Update & Security > Backup`.

### Recovering deleted session transcripts

Session JSONL files are plain text. If they were deleted rather than overwritten:

- On Windows, use `Recuva` and search for `*.jsonl` files in the `CODEX_HOME` path.
- On macOS, check `~/.Trash` and Time Machine if configured.
- On Linux, check whether your filesystem supports `extundelete` or `photorec` for ext4 recovery.

There is no built-in Codex CLI recovery tool. OpenAI has not published a rollback procedure for either issue at the time of writing.[^1][^2]

---

## When Will This Be Fixed?

Issue #36937 remains open as of 4 October 2026.[^1] OpenAI has shipped mitigations for adjacent attack surfaces — dangerous-command detection improvements in v0.147.0, `/diff` hook isolation, and PowerShell classifier scoping — but the specific path that places a JSONL file in a shell program position has not been patched at the platform level.[^4]

The mitigations in this article are preventive. They reduce the probability of triggering the bug to near-zero for normal usage patterns. The config audit checklist and AGENTS.md file rules cost you five minutes to apply. Apply them now.

---

## Summary

| Action | Priority | Time required |
|---|---|---|
| Set `sandbox.mode: standard` in config.yaml | Critical | 2 minutes |
| Disable `fullAccessMode` if enabled | Critical | 1 minute |
| Add file safety rules to AGENTS.md | High | 3 minutes |
| Back up CODEX_HOME | High | 1 minute |
| Add PostToolUse JSONL block hook | Medium | 5 minutes |

If you are on Windows and use Full Access Mode, do the first two items before your next session. The bug is reproducible, the deletion is silent, and there is no built-in recovery path.

---

[^1]: GitHub Issue #36937, "[BUG] Codex executed its own safety instructions from a rollout JSONL, deleting the Windows HOME directory," openai/codex, August 2026. <https://github.com/openai/codex/issues/36937>

[^2]: GitHub Issue #37419, "Data loss: CODEX_HOME runtime directories deleted under a running app on Windows," openai/codex, August 2026. <https://github.com/openai/codex/issues/37419>

[^3]: Codex CLI documentation, "PostToolUse hooks," openai.github.io/codex, 2026. <https://openai.github.io/codex/hooks>

[^4]: Codex Knowledge Base, "Data as Code: How Codex CLI's Own Safety Instructions Became a Confused Deputy Attack on Windows," danielvaughan.com, August 2026. <https://codex.danielvaughan.com/2026/08/18/data-as-code-confused-deputy-codex-cli-rollout-jsonl-safety-instruction-execution-windows-shell-backtick/>
