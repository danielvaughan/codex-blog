---
title: "Codex CLI v0.169.0 Broke Your CI Pipeline — and Said Nothing"
date: 2026-11-17T09:00:00+00:00
last_modified_at: 2026-11-21T10:07:26Z
tags:
  - codex-cli
  - ci-cd
  - mcp
  - trust-model
  - breaking-changes
  - devops
  - v0169
parent: "Articles"
nav_order: 2117
copy_edited: true
rating: 4.0
depth: 4
originality: 4
accuracy: 4
substack: 4
---

Codex CLI v0.169.0 shipped on 13 November with a quiet trust model change that has been silently disabling MCP servers in CI/CD pipelines ever since. No error. No warning. Just MCP integrations that stopped working and green builds that suddenly do less than they used to.

This is an account of what changed, why it happened, how to find out if you are affected, and what to do about it.

---

## What Changed

Before v0.169.0, running `codex exec` with `--sandbox workspace-write` was enough to enable project configuration and MCP server access in an untrusted directory. Developers relied on this combination routinely — it was the standard invocation for headless CI runs where you wanted write access to the workspace without explicitly declaring the directory trusted.

In v0.169.0, that is no longer true. The trust semantics were tightened: `--sandbox workspace-write` now only grants file-write capability. It does not, by itself, signal that the project directory should be trusted for configuration loading or MCP server activation.

To restore the previous behaviour, you must add `--trust` explicitly:

```bash
# Before v0.169.0 (worked implicitly)
codex exec --sandbox workspace-write "run tests and fix failures"

# After v0.169.0 (requires explicit trust declaration)
codex exec --sandbox workspace-write --trust "run tests and fix failures"
```

Without `--trust`, Codex loads in a restricted posture: MCP servers defined in `.codex/config.json` are not started, project-level `AGENTS.md` instruction overrides may not be read, and any tool that depends on project configuration silently falls back to defaults or does nothing.

---

## Why It Is Silent

The failure mode is not an error. When Codex enters an untrusted directory without `--trust`, it does not crash, does not print a warning, and does not exit with a non-zero code because of the trust gap alone. It simply runs in a reduced capability mode.

This means:

- Your CI pipeline passes
- Your Codex invocation completes
- Your MCP-dependent steps produce no output or reduced output
- You have no log line to grep for

The only signal is behavioural: the work that MCP-augmented steps used to do quietly stops happening. If your Codex CI step was running an MCP server to push test results to a dashboard, that push stops. If it was using a custom MCP tool to validate a schema, that validation stops. The build stays green. The gap grows silently.

This is the worst category of breaking change: it affects correctness, not build status.

---

## The Security Rationale Is Sound

It is worth being clear: the underlying change is defensible. Automatically trusting a directory because `--sandbox workspace-write` is set conflates two distinct concerns — what the agent can write, and whether the directory's configuration should be trusted to shape agent behaviour.

An attacker who can place a `.codex/config.json` in a workspace directory could, under the old model, inject MCP server definitions into any `codex exec` run that passed `--sandbox workspace-write`. Requiring explicit `--trust` decouples those two decisions. Operators who want write access but do not want to load project configuration can now have that posture cleanly.

The problem is not the change. The problem is that it was not labelled as a breaking change, and no migration guide was published.

---

## How to Audit Your Pipelines

Search every CI configuration file that invokes `codex exec`:

```bash
# GitHub Actions, CircleCI, GitLab CI, Buildkite
grep -r "codex exec" .github/ .circleci/ .gitlab-ci.yml buildkite/ Makefile scripts/ 2>/dev/null
```

For each match, check whether `--trust` is present. If it is not, and if the invocation relies on MCP servers or project-level AGENTS.md configuration, you are affected.

Also check shell wrapper scripts:

```bash
grep -r "codex exec" --include="*.sh" --include="*.bash" .
```

And any Makefile targets that wrap `codex exec`:

```bash
grep -A3 "codex exec" Makefile
```

The fix is a one-line addition: add `--trust` to every `codex exec` invocation that runs in a directory containing Codex configuration.

---

## The Broader Pattern: Alpha Velocity Meets Production Pipelines

On 17 November 2026 alone, the openai/codex repository shipped two Rust alpha versions (rust-v0.171.0-alpha.11 and alpha.12) on top of the sprint cycle that had been averaging a new alpha every few hours since alpha.6 on 15 November. The pace of development is extraordinary. It is also a source of risk for any team that is not carefully pinning their Codex CLI version.

The v0.169.0 trust change is a symptom of a structural tension: a project moving at alpha velocity while being used in production CI/CD pipelines by thousands of teams. When you update Codex CLI without reading the full release notes — and the release notes for v0.169.0 did not explicitly mark this as a breaking change — you are exposed.

Three practices mitigate this:

**1. Pin your Codex CLI version in CI.** Do not use `@latest`. Use a specific version tag in your package manager configuration, your Docker image, or your Homebrew pin file. Treat it like any other production dependency.

```bash
# npm / package.json
"@openai/codex": "0.169.1"

# Homebrew (manual pin)
brew pin codex

# Docker
FROM openai/codex:0.169.1
```

**2. Run Codex CLI smoke tests in CI.** A five-line test that invokes `codex exec --trust` in a minimal project directory and checks that an expected MCP tool is called would have caught this regression immediately. It costs minutes to write and seconds to run.

```bash
# Example smoke test
output=$(codex exec --sandbox workspace-write --trust "list available tools" 2>&1)
echo "$output" | grep -q "my-mcp-tool" || { echo "MCP server not loaded — trust flag issue?"; exit 1; }
```

**3. Watch the CHANGELOG before updating.** The openai/codex CHANGELOG is maintained on GitHub. Subscribe to releases. Before updating in a production context, scan the diff between your current version and the new version for any changes to `exec`, `sandbox`, or `trust` semantics.

---

## Recommended Immediate Actions

If you are running Codex CLI in CI/CD pipelines:

1. **Audit** all `codex exec` invocations using the grep commands above
2. **Add `--trust`** to any invocation that relies on MCP servers or project configuration
3. **Pin** your Codex CLI version to prevent silent future upgrades
4. **Write a smoke test** that verifies MCP tools load correctly in your CI environment
5. **Check your MCP tool outputs** from the last two weeks — if they went quiet on or after 13 November, you now know why

---

## The Release Discipline Gap

The community response to this incident has been largely measured, but the underlying frustration is legitimate. One user summarised it cleanly:

> "My CI was green for three weeks. I changed nothing. v0.169.0 dropped and my MCP integrations silently stopped working. This should have been a major version bump."

The security improvement is real. The user experience of discovering it via silent pipeline degradation is not acceptable for a tool that positions itself as production-ready. Breaking changes to trust semantics — particularly ones that cause silent correctness failures rather than visible errors — warrant an explicit `BREAKING:` label in the CHANGELOG, a migration note in the release announcement, and ideally a deprecation warning during the transition period.

As Codex CLI moves toward the v0.171.0 stable cycle (alpha.12 as of 17 November, stable expected this week), the team has an opportunity to establish a stronger release discipline norm: `BREAKING:` labels are not optional for changes that silently alter agent behaviour in production environments.

Until that norm is established, the responsibility falls to teams using Codex in CI/CD to treat every minor version bump as a potential behaviour change, audit actively, and pin defensively.

---

## Summary

| Item | Detail |
|------|--------|
| **Affected version** | v0.169.0 (released 13 November 2026) |
| **What changed** | `--sandbox workspace-write` no longer enables project config / MCP servers in untrusted dirs |
| **Fix** | Add `--trust` to `codex exec` invocations |
| **Failure mode** | Silent — no error, no non-zero exit code |
| **Detection** | Behavioural: MCP-dependent steps stop producing expected outputs |
| **Prevention** | Pin Codex CLI version; add MCP smoke tests to CI |

---

*Sources: community.openai.com/c/codex (threads fetched 2026-11-17); reddit-sentiment/2026-11-17-daily-report.md; github.com/openai/codex release notes v0.169.0; openai/codex GitHub discussions (Nov 13–17 2026).*
