---
title: "AGENTS.md: The 4 Traps That Silently Break Your Instructions in Codex, Claude Code, Cursor, and Gemini CLI"
date: 2026-11-16T13:00:00+00:00
last_modified_at: 2026-11-19T10:27:36Z
parent: "Articles"
nav_order: 2111
tags: ["AGENTS.md", "codex-cli", "claude-code", "cursor", "gemini-cli", "configuration", "cross-tool", "silent-failure", "best-practices"]
copy_edited: true
---

# AGENTS.md: The 4 Traps That Silently Break Your Instructions in Codex, Claude Code, Cursor, and Gemini CLI


---

The promise of `AGENTS.md` is appealing: one file, any agent, consistent instructions across your whole team. In practice, each tool has at least one silent failure mode — a misconfiguration that causes your instructions to be partially or completely ignored, without any error message telling you something went wrong.

This article documents four of them — one per major tool — and gives you the verification command to confirm whether your instructions are actually being read.[^1]

---

## Trap 1 (Codex CLI): An `agents.override.md` Anywhere in the Tree Silently Wins

Codex CLI supports an override mechanism via `agents.override.md`. This file, if present in *any* subdirectory of your project, supersedes the root-level `AGENTS.md` for tasks that touch that directory — without warning, without a log message, without any visible indication.

The practical failure mode: a developer adds an `agents.override.md` to a module or service subfolder for a one-off customisation, forgets about it, and later your root-level `AGENTS.md` updates simply stop applying in that scope. Codex reads the override silently and proceeds. You notice inconsistent behaviour but no error trail to follow.

**Compounding factor:** Codex CLI's total context budget for instruction files is capped at **32 KB**. Beyond that limit, additional content is silently truncated — again without warning. A large `AGENTS.md` padded with architecture diagrams, generated summaries, or verbose examples can breach this limit and cause the second half of your instructions to never be read.

**Verification command:**

Ask Codex directly:

```text
summarise current instructions
```

Codex will render the full effective instruction context it is operating under for the current session. If a section of your `AGENTS.md` is missing, an override is active, or a truncation has occurred, the gap will be visible in the summary.

---

## Trap 2 (Claude Code): A Local `claude.md` or `claude.local.md` File Disables AGENTS.md Entirely

Claude Code does not read `AGENTS.md` natively. Its native format is `CLAUDE.md`, and its fallback is `AGENTS.md` — but only under a specific condition: **no `claude.md` or `claude.local.md` file exists anywhere in the directory tree**.[^2]

The moment a local `claude.local.md` appears — even an empty one, even one added for a different purpose — Claude Code switches back to its own instruction format and stops consulting `AGENTS.md`. This is the most common failure mode in multi-tool teams: someone creates a `claude.local.md` for a quick one-off override during a session, commits it to source control, and your shared `AGENTS.md` goes dark for Claude Code users from that point forward.

The same trap applies in the other direction. Teams that invest heavily in `CLAUDE.md` and expect Claude Code to also respect `AGENTS.md` are disappointed to find the two are not read simultaneously — Claude Code reads one or the other, not both.

**Verification command:**

```text
/status
```

The `/status` output in Claude Code includes the active instruction source. If it lists `CLAUDE.md` or `claude.local.md` rather than `AGENTS.md`, you know which file is in control.

---

## Trap 3 (Cursor): Plain `.md` Files in `.cursor/rules/` Are Ignored

Cursor supports scoped, per-glob instruction files in `.cursor/rules/`. This is a genuinely powerful feature — different rules for different parts of the codebase, applied only when the matching files are in context. The trap: **plain `.md` files in that directory are silently ignored**. Cursor only reads `.mdc` files (Markdown-with-frontmatter), and only when they carry specific frontmatter fields.

A valid Cursor rule file requires `.mdc` extension and at least one of the following frontmatter fields:

```yaml
---
alwaysApply: true        # apply this rule in every session
globs: ["src/**/*.ts"]   # apply when these files are in context
description: "..."       # apply when this rule is semantically relevant
---
```

Without these fields — or with the file saved as `.md` instead of `.mdc` — the file exists in your `.cursor/rules/` directory but contributes nothing to any agent session. The failure is especially insidious because the file appears valid: it's markdown, it's in the right folder, it looks like it should work.

**Verification command:**

In Cursor, open the Model Context panel (⌘M) and inspect the **Rules** section. Each `.mdc` file that is actively loaded for the current context will appear there. Files not listed are either not `.mdc`, missing required frontmatter, or scoped to globs that don't match the current open files.

---

## Trap 4 (Gemini CLI): The Default Read Target Is `gemini.md`, Not `agents.md`

Gemini CLI does not read `AGENTS.md` by default. Its default instruction file is `gemini.md` (lowercase, in the project root). If your team has invested in an `AGENTS.md` and added Gemini CLI to the workflow, the agent runs with no project instructions until someone explicitly configures the read target.

The fix is a small change in `.gemini/settings.json` in the project root:

```json
{
  "context": {
    "fileName": ["AGENTS.md"]
  }
}
```

Once set, Gemini CLI reads `AGENTS.md` as its instruction source. Without this change, Gemini silently operates in a context-free mode — no error, no warning, just no instructions applied.

**Verification command:**

```text
/context
```

The `/context` command in Gemini CLI lists all files currently loaded into the agent's context window. If `AGENTS.md` does not appear in the list, the `settings.json` change has not been applied (or the file path is wrong).

---

## The Cross-Tool Setup Checklist

Each trap has a corresponding check. Running all four takes under 10 minutes and surfaces every active instruction source across your toolchain:

| Tool | Check command | What to look for |
|---|---|---|
| Codex CLI | `summarise current instructions` | Confirm full AGENTS.md content; look for unexpected override sections |
| Claude Code | `/status` | Confirm active instruction source is AGENTS.md, not claude.local.md |
| Cursor | Model Context panel → Rules | Confirm .mdc files are listed; check globs match open files |
| Gemini CLI | `/context` | Confirm AGENTS.md appears in loaded files list |

Run this checklist when onboarding a new team member, after any change to instruction files, and whenever agent behaviour diverges unexpectedly from your documented instructions.

---

## Bonus: Claude Code's `/doctor prompt-audit` for Stale Instructions

Since v2.1.283, Claude Code ships a built-in instruction auditor accessible via `/doctor prompt-audit`.[^3] It scans your active instruction file (whether `CLAUDE.md` or `AGENTS.md`) for three categories of technical debt:

1. **Instructions written for older model versions** — directives that reference deprecated behaviour or capabilities that no longer apply to the current Claude model.
2. **Dead file references** — `@import` paths or file citations that point to files that have been renamed, moved, or deleted.
3. **Contradictions between instructions** — where two rules apply to the same scenario but specify different behaviour.

The audit output includes a severity rating and suggested remediation for each finding. For teams with mature instruction files that have accumulated over many months of iteration, running `/doctor prompt-audit` once per quarter is a low-effort way to prevent instruction debt from quietly degrading agent performance.

---

## The Deeper Principle

Each of these traps shares a structural property: the failure is invisible. No red error message, no session warning, no log entry pointing to what went wrong. The agent simply behaves as if your instructions don't exist — or as if a different set of instructions were active. That invisibility is what makes them costly: debugging inconsistent agent behaviour when no errors are surfaced requires knowing where to look.

The verification commands above give you exactly that: a direct window into what each tool is actually reading. Use them proactively rather than reactively, and the traps become maintenance checkpoints rather than incident triggers.

---

[^1]: Primary source: "AGENTS.md Explained" — Daily AI News By Ali (youtube.com/watch?v=m6NmVmrA24c, 2026-11-16, 10:32). Tool-specific behaviour confirmed against official documentation for each agent.
[^2]: Claude Code AGENTS.md fallback behaviour: documented in Claude Code release notes and official Anthropic developer documentation. The fallback is intentional — it allows teams without a `CLAUDE.md` to benefit from AGENTS.md without additional configuration.
[^3]: `/doctor prompt-audit` (alias: `/checkup prompt-audit`) available in Claude Code v2.1.283 and later. Feature referenced in Claude Code official documentation (Anthropic, 2026). The command name derives from "Doctor Prompt" — a reference to the medical audit framing used in Claude Code's instruction hygiene documentation.
