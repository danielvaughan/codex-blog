---
title: "Codex in the Cloud: AGENTS.md Patterns That Travel"
parent: "Articles"
nav_order: 1228
date: 2026-11-11T09:00:00+00:00
last_modified_at: 2026-11-11T13:04:54Z
tags: ["codex-cli", "codex-cloud", "agents-md", "cloud", "configuration", "secret-management", "session-continuity", "devday-2026"]
---

# Codex in the Cloud: AGENTS.md Patterns That Travel


---

OpenAI DevDay 2026 confirmed what had been hinted at across several preview releases: Codex Cloud is production-ready.[^1] Tasks run remotely on OpenAI infrastructure, your laptop can close, and work continues. You can hand off a session from a MacBook to a Linux workstation without losing state. The cloud execution model removes the main constraint that kept Codex CLI a local tool — the requirement to keep a machine running and connected.

The shift introduces three structural differences from local Codex CLI sessions that most AGENTS.md files are not written to handle: restricted filesystem access, a changed secret management model, and a session continuity mechanism that works differently to local worktrees. An AGENTS.md file written for local execution will not break in Codex Cloud, but it will under-perform. This article documents what changes, why it matters, and how to write AGENTS.md patterns that work reliably in both environments.

## What Codex Cloud Actually Is

Codex Cloud is not a hosted version of the CLI you run locally. It is a managed execution environment where each task runs in an isolated container with a fresh clone of your repository.[^2] The container has network access to the repositories and secrets you have granted, and nothing else.

Three constraints follow directly from this architecture:

1. **No local filesystem paths.** The container does not have access to your home directory, local config files, or paths outside the repository. References like `~/.codex/memories/` or `/usr/local/lib/node_modules/` will fail silently or raise errors.
2. **No `.env` files in the working directory.** Local secret injection via `.env` is a local pattern. Codex Cloud uses environment variables injected at task creation time, managed through the OpenAI Codex Cloud dashboard or the `codex cloud env set` command.
3. **Sessions are named, not path-bound.** Local Codex CLI uses git worktrees to isolate parallel sessions. Codex Cloud uses named sessions with server-side state. You resume a session by name, not by switching to a different directory.

None of these constraints are new to cloud execution generally. What is new is that Codex CLI users who have built up AGENTS.md files over months of local development are now deploying those files into an environment that silently ignores some of their instructions.

## The Filesystem Problem

The most common local AGENTS.md pattern that fails in Codex Cloud is any instruction that references an absolute path outside the repository.

Examples that break:

```markdown
## Memory
Read `~/.codex/memories/memory_summary.md` at session start.
Store session summaries to `~/.codex/memories/rollout_summaries/`.
```

```markdown
## Linting
Run `node /usr/local/lib/node_modules/.bin/eslint` before committing.
```

```markdown
## Secrets
Load credentials from `/home/daniel/.config/project/credentials.json`.
```

In a local session, these paths resolve. In Codex Cloud, they do not exist. The agent will attempt the operation, fail, and either raise an error or silently skip it — depending on the instruction phrasing.

**The fix** is to make all filesystem references repository-relative:

```markdown
## Memory
Read `.codex/session-context.md` at session start if it exists.
Store summaries to `.codex/summaries/` using ISO date filenames.
```

```markdown
## Linting
Run `npx eslint` or `./node_modules/.bin/eslint` — do not use absolute paths.
```

The `.codex/` directory pattern is worth adopting as a convention. It is git-ignorable via `.gitignore`, portable across environments, and consistent with how the CLI itself manages per-repository state.[^3]

## The Secret Management Problem

Local Codex CLI workflows commonly load secrets from a `.env` file in the working directory or from the host environment. Neither approach transfers cleanly to Codex Cloud.

Codex Cloud injects secrets as environment variables at the container level, configured before the task runs. The implication is that your AGENTS.md should reference secrets by environment variable name rather than by file path or `.env` key.

AGENTS.md that breaks in Codex Cloud:

```markdown
## API Keys
Load `OPENAI_API_KEY` from `.env` in the project root.
If `.env` is absent, prompt for the key.
```

AGENTS.md that works in both environments:

```markdown
## API Keys
Read `OPENAI_API_KEY` from the environment. Do not load it from files.
If the variable is absent, halt and report which key is missing — do not prompt.
```

The "halt and report" instruction is deliberate. In a local session, a missing key is an operator error that can be fixed interactively. In a Codex Cloud task running unattended, prompting is not possible. Failing fast with a clear error message is better than producing a partial result or hanging.

For secrets that must exist in the repository for tooling to work (for example, a `.npmrc` referencing an authentication token), use Codex Cloud's secret interpolation syntax in the file rather than storing the token directly:

```ini
//registry.npmjs.org/:_authToken=${NPM_TOKEN}
```

The `NPM_TOKEN` variable is then set in Codex Cloud's environment configuration, not in the file.[^2]

## The Session Continuity Problem

Local Codex CLI uses git worktrees for parallel sessions. Each worktree is a separate directory with its own branch; you switch between sessions by switching directories. Codex Cloud uses named sessions instead.

A named session persists across container restarts, device switches, and time zones. You create it once and resume it by name:

```bash
codex cloud session create --name "auth-refactor" --repo my-org/my-repo
codex cloud session resume auth-refactor
```

The implication for AGENTS.md is that instructions written around worktree conventions do not apply in Codex Cloud, and instructions that assume a single active session per machine are too restrictive.

The practical patterns to adopt:

**Avoid worktree-specific instructions.** Instructions like "create a new worktree for each feature branch" describe a local execution pattern. In Codex Cloud, session isolation is handled at the container level.

**Name sessions by task, not by time.** The session name is the primary handle for resumption and handoff. Names like `auth-refactor-nov11` are more useful than `session-1` or `feature-work`.

**Write AGENTS.md to tolerate cold starts.** In a local session that runs for hours, the agent accumulates context. In a Codex Cloud session resumed after the container has been recycled, the agent starts fresh from AGENTS.md plus any state stored in the repository. AGENTS.md should therefore contain enough context for the agent to orient itself without relying on in-memory state from a prior run.

## A Cloud-Compatible AGENTS.md Template

The following template consolidates the patterns above into a structure that works in both local and cloud environments. The key design decisions are: repository-relative paths throughout, environment variable references for secrets, no worktree instructions, and explicit cold-start orientation.

```markdown
# AGENTS.md

## Project
Name: [project name]
Repo: [org/repo]
Primary language: [language]
Package manager: [npm / pip / cargo / etc]

## Orientation
At session start:
1. Run `git log --oneline -5` to understand recent work.
2. Read `.codex/session-context.md` if it exists.
3. Read `CHANGELOG.md` or `RELEASES.md` if present.
Do not assume you have memory of prior sessions.

## Environment
Secrets come from environment variables only. Do not read from `.env`.
Required variables: [list each one and its purpose]
If any required variable is absent, halt immediately and list what is missing.

## Filesystem
All file paths must be repository-relative.
Store session notes to `.codex/summaries/YYYY-MM-DD.md`.
Never reference paths outside the repository root.

## Code quality
Run `[lint command]` before committing.
Run `[test command]` before marking a task complete.
Never commit if tests fail.

## Commits
Follow Conventional Commits: `type(scope): description`.
Commit individual logical changes — do not batch unrelated edits.

## Communication
When a task completes, write a one-paragraph summary to `.codex/summaries/YYYY-MM-DD.md`.
Include: what was done, what was skipped, and any decisions made.
```

The `.codex/summaries/` directory pattern deserves emphasis. It is the cloud-compatible replacement for `~/.codex/memories/`. Because it lives inside the repository, summaries survive container recycling and are visible across devices. Add `.codex/` to `.gitignore` to keep these files out of commits, or omit them from `.gitignore` if you want session context to persist in version control.[^3]

## Auditing an Existing AGENTS.md for Cloud Compatibility

If you have an existing AGENTS.md built for local use, the audit is mechanical. Check each instruction against three questions:

1. Does this reference an absolute path? If yes, make it repository-relative or remove it.
2. Does this reference a file for secret loading (`.env`, credential files, config paths)? If yes, replace with environment variable references.
3. Does this assume persistent in-memory state from a prior session? If yes, rewrite it to treat every session as a cold start.

The majority of AGENTS.md files that have evolved organically from local use will fail on points one and three. Point two is less common because most teams already use environment variables for secret injection, but it appears frequently in older files written before `.env` practices stabilised.

## What Does Not Change

Cloud execution does not change the AGENTS.md features that matter most: tool permissions, approval policies, session profiles, and task-level instruction scoping. The `writable_roots`, `network`, and `approval_policy` keys work identically in Codex Cloud.[^2] Session profiles defined in `config.toml` apply whether the session runs locally or in a container.

The `/agents` view introduced in Codex CLI v0.157 and improved in v0.159 also works with Codex Cloud sessions — you can monitor and interrupt cloud tasks from the same TUI you use to manage local sessions.[^4]

The main adjustment is the filesystem and secret model. Get those right, and an AGENTS.md file that works locally will work in Codex Cloud without further modification.

---

[^1]: OpenAI DevDay 2026 — Codex Cloud production announcement. 29 September 2026, San Francisco.
[^2]: Codex Cloud documentation — container environment, secret injection, named sessions. `https://developers.openai.com/codex/models`.
[^3]: Codex CLI repository conventions — `.codex/` directory as a portable per-repo state location.
[^4]: Codex CLI v0.159.0 release notes — `/agents` view improvements and cloud session monitoring.
[^5]: The /agents View: Managing Parallel Codex Agents from One CLI Session. `2026-10-17-agents-view-parallel-task-management-codex-cli-tui.md`.
