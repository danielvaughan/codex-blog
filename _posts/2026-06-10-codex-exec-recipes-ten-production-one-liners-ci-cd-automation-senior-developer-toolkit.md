---
title: "Ten codex exec One-Liners Every Senior Developer Should Have in Their Shell History"
type: Technical Article
timestamp: 2026-06-10T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-06-10-codex-exec-recipes-ten-production-one-liners-ci-cd-automation-senior-developer-toolkit"
tags: ["codex-cli", "codex-exec", "automation", "ci-cd", "one-liners", "non-interactive", "structured-output", "shell-scripting", "developer-productivity"]
date: 2026-06-10T09:00:00+00:00
last_modified_at: 2026-09-03T02:12:24+01:00
---
# Ten `codex exec` One-Liners Every Senior Developer Should Have in Their Shell History


---

`codex exec` is the non-interactive entry point to Codex CLI — it runs a single task, streams progress to stderr, prints the result to stdout, and exits [^1]. It is the primitive that turns Codex from an interactive pair programmer into a scriptable automation engine. Yet most developers use it for ad-hoc prompts and never build it into their daily workflow.

This article presents ten production-tested `codex exec` commands that solve real problems senior developers face every week. Each recipe includes the exact invocation, an explanation of the flags, and notes on where it fits — shell alias, Git hook, CI job, or cron task.

## 1. Triage a Failing CI Log

When a CI pipeline fails at 02:00 and you need a root-cause summary before your first coffee:

```bash
curl -sL "$CI_JOB_LOG_URL" | \
  codex exec "Identify the root cause of this CI failure. List the failing test or build step, the error message, and a concrete fix. Ignore warnings." \
  -s read-only --ephemeral
```

The `-s read-only` sandbox prevents any file writes — you want analysis, not action [^2]. The `--ephemeral` flag skips persisting session state to disk, keeping your sessions directory clean [^1].

**Where it fits:** GitLab CI `after_script`, GitHub Actions failure step, or a shell alias wrapping `gh run view --log-failed`.

## 2. Generate a Structured PR Description from a Diff

```bash
git diff main...HEAD | \
  codex exec "Write a pull request description for this diff. Include a Summary section (3 bullets max), a Testing section listing what was verified, and a Risk Assessment section rating risk as Low/Medium/High with justification." \
  -s read-only --ephemeral -o /tmp/pr-body.md
```

The `-o` flag writes the final message to a file while still printing to stdout [^1]. Pipe the result straight into `gh pr create --body-file /tmp/pr-body.md`.

**Where it fits:** A `pr-create` shell function or a Git `prepare-commit-msg` hook.

## 3. Audit Dependencies for Known Vulnerabilities

```bash
codex exec "Run 'npm audit --json' (or the equivalent for this project's package manager), parse the output, and produce a markdown table of Critical and High severity vulnerabilities with columns: Package, Severity, CVE, Fix Version. If no vulnerabilities are found, say so." \
  -s read-only --ephemeral
```

By keeping the sandbox read-only, Codex can run `npm audit` but cannot modify `package-lock.json` — the fix decision stays with you [^2].

**Where it fits:** Weekly cron job that posts results to a Slack channel via `curl`.

## 4. Detect Dead Exports Across a TypeScript Project

```bash
codex exec "Find all exported symbols in this TypeScript project that are never imported by any other file. List each dead export with its file path and line number. Ignore index.ts barrel files." \
  -s read-only --ephemeral --json 2>/dev/null | \
  jq -r 'select(.type == "item.completed") | .item.content // empty'
```

The `--json` flag emits every Codex event as a JSON Lines stream [^1]. Piping through `jq` extracts only the final agent message, which integrates cleanly into existing shell pipelines.

**Where it fits:** Pre-merge CI check. Fail the pipeline if the output is non-empty.

## 5. Scan a Diff for Accidental Secret Exposure

```bash
git diff --cached | \
  codex exec "Inspect this staged diff for accidentally committed secrets: API keys, tokens, passwords, private keys, connection strings, or .env values. For each finding, report the file, line, and the type of secret. If none found, respond with 'CLEAN'." \
  -s read-only --ephemeral
```

This is a pre-commit gate. Wire it into a Git `pre-commit` hook so it runs before every commit [^3]:

```bash
#!/usr/bin/env bash
result=$(git diff --cached | codex exec "..." -s read-only --ephemeral 2>/dev/null)
if [ "$result" != "CLEAN" ]; then
  echo "Potential secret detected:" >&2
  echo "$result" >&2
  exit 1
fi
```

**Where it fits:** `.git/hooks/pre-commit` or a shared Codex CLI plugin with bundled hooks [^4].

## 6. Generate Release Notes from Git History

```bash
codex exec "Read the Git log between the two most recent tags. Categorise commits into Features, Fixes, and Chores. Write concise release notes in Keep a Changelog format. Exclude merge commits and dependabot bumps." \
  -s read-only --ephemeral -o RELEASE_NOTES.md
```

The structured output lands in `RELEASE_NOTES.md` ready for `gh release create --notes-file RELEASE_NOTES.md` [^1].

**Where it fits:** Release automation script, typically triggered by a tag push in CI.

## 7. Produce a Structured Migration Inventory

When you need to find every call site of a deprecated API before a migration deadline:

```bash
codex exec "Find every import and usage of the Assistants API (threads.create, threads.runs.create, threads.messages.list, client.beta.assistants) in this repository. Output a JSON array where each object has 'file', 'line', 'api_call', and 'migration_complexity' (simple|moderate|complex)." \
  -s read-only --ephemeral \
  --output-schema migration-schema.json \
  -o migration-inventory.json
```

The `--output-schema` flag constrains the model's final response to a JSON Schema you provide, guaranteeing machine-parseable output [^1]. For the schema file:

```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "file": { "type": "string" },
      "line": { "type": "integer" },
      "api_call": { "type": "string" },
      "migration_complexity": { "type": "string", "enum": ["simple", "moderate", "complex"] }
    },
    "required": ["file", "line", "api_call", "migration_complexity"]
  }
}
```

**Where it fits:** Pre-migration planning. Feed the output into a tracking spreadsheet or Jira bulk-import [^5].

## 8. Validate AGENTS.md Against Codex Conventions

```bash
codex exec "Read every AGENTS.md file in this repository. Check each for: (1) instructions that contradict each other across directory layers, (2) prose paragraphs longer than 5 lines that should be bullet points, (3) references to models or features that no longer exist in Codex CLI v0.139, (4) missing explicit priority ordering when multiple rules could conflict. Report findings as a numbered list." \
  -s read-only --ephemeral
```

AGENTS.md files accumulate drift over time — instructions written for v0.120 may reference deprecated features or use patterns that no longer work [^6]. Running this check weekly catches configuration rot before it silently degrades agent behaviour.

**Where it fits:** Weekly scheduled CI job or a `make lint-agents` target.

## 9. Benchmark Token Cost of an MCP Server

```bash
codex exec "List every MCP tool registered in this session, its name, and an estimate of its JSON Schema definition size in tokens. Sort by token cost descending. Calculate the total schema overhead per turn." \
  -s read-only --ephemeral --json 2>/dev/null | \
  jq -r 'select(.type == "item.completed") | .item.content // empty'
```

The MCP tax — tool definition tokens injected on every turn — is the single largest hidden cost in most Codex CLI workflows [^7]. This recipe surfaces exactly how much each MCP server contributes so you can scope `enabled_tools` in your `config.toml`:

```toml
[mcp_servers.github]
command = "gh-mcp"
enabled_tools = ["create_pull_request", "list_issues", "get_file_contents"]
```

Restricting to only the tools you need can reduce per-turn overhead from 55,000 tokens to under 5,000 [^7].

**Where it fits:** Run once after adding a new MCP server, then again monthly to audit drift.

## 10. Resumable Multi-Step Pipeline

`codex exec resume` continues a previous non-interactive session, preserving context across pipeline stages [^1]:

```bash
# Stage 1: Analyse
codex exec "Analyse this repository for accessibility violations against WCAG 2.2 AA. Write findings to a11y-report.md." \
  -s workspace-write --ephemeral

# Stage 2: Fix (resumes context from stage 1)
codex exec resume --last "Now fix the Critical and High severity findings you identified. Do not modify any test files." \
  -s workspace-write

# Stage 3: Verify (resumes context from stage 2)
codex exec resume --last "Run the project's test suite and verify all a11y fixes pass. Summarise results." \
  -s workspace-write
```

Each `resume --last` picks up where the previous `codex exec` left off, carrying the full conversation history [^1]. This pattern avoids re-analysing the codebase on every stage while keeping each step's permissions scoped appropriately.

**Where it fits:** Complex CI pipelines where analysis, remediation, and verification are separate jobs sharing a persistent workspace.

## Composition Patterns

These recipes compose. A morning automation script might chain several:

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "=== Dependency Audit ===" >&2
codex exec "..." -s read-only --ephemeral

echo "=== Dead Export Check ===" >&2
codex exec "..." -s read-only --ephemeral

echo "=== AGENTS.md Lint ===" >&2
codex exec "..." -s read-only --ephemeral
```

Each invocation is independent — no shared state, no side effects, no leaked context [^1]. Codex CLI's process model makes this safe: each `codex exec` starts a fresh session unless you explicitly `resume`.

## Authentication in CI

For CI/CD pipelines, use `CODEX_API_KEY` for API-key authentication [^1]:

{% raw %}
```bash
CODEX_API_KEY=${{ secrets.OPENAI_API_KEY }} codex exec "..." --ephemeral
```
{% endraw %}

For ChatGPT workspace authentication with per-user audit attribution, use v2 personal access tokens [^8]:

```bash
printenv CODEX_ACCESS_TOKEN | codex login --with-access-token
codex exec "..." --ephemeral
```

⚠️ `CODEX_API_KEY` only works with `codex exec`, not with interactive `codex` sessions [^1].

## Profile-Based CI Configuration

Keep CI-specific settings in a dedicated profile rather than scattering flags across every invocation [^9]:

```toml
# ~/.codex/ci.config.toml
model = "gpt-5.5"
approval_policy = "never"
sandbox_mode = "workspace-write"
model_reasoning_effort = "medium"

[mcp_servers.github]
command = "gh-mcp"
enabled_tools = ["create_pull_request", "list_issues"]
```

Then every CI recipe simplifies to:

```bash
codex exec --profile ci "your task" --ephemeral
```

## What to Watch

Two known limitations apply to `codex exec` pipelines as of v0.139:

1. **`--output-schema` with active MCP servers** can produce malformed JSON when tools inject unexpected output into the completion context [^10]. Test schema-constrained recipes with and without MCP servers enabled.

2. **`codex exec` OTel metrics** are incomplete — the `[otel]` configuration in `config.toml` is fully respected only by the interactive CLI, while `codex exec` has gaps in metrics export [^11]. If you need per-pipeline token attribution, parse the `--json` event stream instead.

## Citations

[^1]: OpenAI, "Non-interactive mode – Codex", [https://developers.openai.com/codex/noninteractive](https://developers.openai.com/codex/noninteractive)

[^2]: OpenAI, "Command line options – Codex CLI", [https://developers.openai.com/codex/cli/reference](https://developers.openai.com/codex/cli/reference)

[^3]: OpenAI, "Best practices – Codex", [https://developers.openai.com/codex/learn/best-practices](https://developers.openai.com/codex/learn/best-practices)

[^4]: OpenAI, "Agent Skills – Codex", [https://developers.openai.com/codex/skills](https://developers.openai.com/codex/skills)

[^5]: OpenAI, "Use Codex CLI to automatically fix CI failures", [https://developers.openai.com/cookbook/examples/codex/autofix-github-actions](https://developers.openai.com/cookbook/examples/codex/autofix-github-actions)

[^6]: Blake Crosley, "AGENTS.md Patterns: What Actually Changes Agent Behavior", [https://blakecrosley.com/blog/agents-md-patterns](https://blakecrosley.com/blog/agents-md-patterns)

[^7]: Scalekit, "MCP Tax: Shell CLI vs MCP Token Cost Comparison", referenced in Codex CLI token consumption community benchmarks, June 2026

[^8]: OpenAI, "Access tokens – Codex", [https://developers.openai.com/codex/enterprise/access-tokens](https://developers.openai.com/codex/enterprise/access-tokens)

[^9]: OpenAI, "Advanced Configuration – Codex", [https://developers.openai.com/codex/config-advanced](https://developers.openai.com/codex/config-advanced)

[^10]: GitHub Issue #15451, "--json and --output-schema are silently ignored when tools/MCP servers are active", [https://github.com/openai/codex/issues/15451](https://github.com/openai/codex/issues/15451)

[^11]: GitHub Issue #12913, "codex exec emits no OTel metrics; codex mcp-server emits no OTel telemetry at all", [https://github.com/openai/codex/issues/12913](https://github.com/openai/codex/issues/12913)
