---
title: "Beyond the Prompt: Codex CLI Mastery"
description: "Most developers treat Codex CLI as a chat box. The real value sits past the prompt, in AGENTS.md, skills, subagents, profiles, MCP servers and directory layout. This guide covers everything between installation and genuine mastery."
date: 2026-05-29T08:00:00+00:00
last_modified_at: 2026-05-29T10:26:09+01:00
tags:
  - mastery
  - agents-md
  - skills
  - subagents
  - mcp
  - config-toml
  - profiles
  - codex-cli
---

![Sketchnote diagram for: Beyond the Prompt: Codex CLI Mastery](/sketchnotes/articles/2026-05-29-codex-cli-mastery-beyond-the-prompt.png)

# Beyond the Prompt: Codex CLI Mastery


Most developers install Codex CLI, type a prompt and wait. When the output disappoints, they write a longer prompt. This is the wrong lever. The difference between a developer who gets passable suggestions and one who ships entire features hands-free is not prompt quality. It is infrastructure: directory layout, AGENTS.md, skills, subagents, profiles, MCP servers and verification loops. This guide covers everything between installation and genuine mastery. *Current as of v0.135.0, May 2026.*

## The single most important principle

Give Codex a way to verify its own work.

Without a feedback loop, you are the only signal. With one, Codex iterates until the code passes. A single line in AGENTS.md can transform output quality by two to three times:

```markdown
After every code change, run `npm test` and fix failures before responding.
```

That is not a prompt trick. It is a structural decision that turns Codex from a suggestion engine into a self-correcting agent. Every tactic in this guide follows the same logic: move intelligence from your prompts into your project configuration, where it compounds across sessions rather than evaporating at the end of each one.

## AGENTS.md: the operating system for your agent

AGENTS.md is the most important file in a Codex CLI project. It loads at session start and ships with every API call as part of the system prompt[^1]. Think of it as the `.editorconfig` of the agentic era, except instead of tab width it governs reasoning, tool use, coding conventions and verification behaviour.

### The discovery hierarchy

Codex walks the filesystem on every run[^1]:

```
1. ~/.codex/AGENTS.override.md    ← highest-precedence global override
   OR ~/.codex/AGENTS.md          ← standard global defaults

2. <git-root>/AGENTS.override.md  ← repo override
   OR <git-root>/AGENTS.md        ← repo-level instructions

3. <subdirectory>/AGENTS.md       ← directory-scoped rules
   (repeated for every subdirectory Codex enters)
```

Override files take precedence at each tier. Subdirectory files are additive: they refine the instructions above them rather than replacing them.

### What belongs in AGENTS.md

Brevity matters. Every token in AGENTS.md is sent with every API call. Include only rules that would cause Codex to produce wrong output if removed[^2]:

- **Build and test commands.** `Run pytest -x after changes. Run mypy --strict before committing.`
- **Package manager.** `Use pnpm, not npm or yarn.`
- **Architecture constraints.** `All API handlers live in src/handlers/. Never import from src/internal/ outside that package.`
- **Type system distinctions.** `This project uses strict TypeScript. Never use any.`
- **Convention traps.** `Date fields use ISO 8601 with timezone. Never store epoch milliseconds.`

General programming knowledge does not belong here, nor do style preferences your linter already enforces. If ESLint catches it, AGENTS.md does not need to say it.

### AGENTS.local.md: your private layer

`AGENTS.local.md` sits alongside `AGENTS.md` but is gitignored[^1]. Use it for personal feedback loops. When a PR reviewer corrects the same mistake repeatedly, say you keep forgetting migration scripts, add it here. Over weeks, the local file accumulates the reviewer's patterns and Codex applies them automatically.

```markdown
<!-- AGENTS.local.md -->
- When adding a new database column, always generate a migration file.
- My PR reviewer insists on explicit return types for all exported functions.
- Prefer early returns over nested conditionals.
```

## The .codex directory

The `.codex` directory at project root is the organisational hub for everything beyond AGENTS.md[^3]:

```
.codex/
├── config.toml          # Project-level configuration
├── skills/              # Reusable prompt-based capabilities
│   ├── tdd/
│   │   └── SKILL.md
│   └── review/
│       └── SKILL.md
├── agents/              # Subagent definitions (TOML)
│   └── security-audit.toml
└── hooks/               # Event-driven automation
    ├── pre-exec.sh
    └── post-patch.sh
```

Each subdirectory serves a distinct purpose. Understanding all four, skills, subagents, hooks and config, is what separates casual use from mastery.

## Skills: the unit of reusable expertise

A skill is a directory containing a `SKILL.md` file[^4]. The Agent Skills standard, governed at agentskills.io, is supported by more than 30 platforms including Codex CLI, Claude Code, Gemini CLI and Cursor[^4]. Any task you perform daily warrants conversion to a skill.

### Anatomy of a skill

```
.codex/skills/tdd/
├── SKILL.md           # Instructions and metadata
└── templates/
    └── test-scaffold.ts
```

The `SKILL.md` front matter declares when the skill activates:

```markdown
---
name: tdd
description: Enforce test-driven development workflow
trigger: when the user asks to implement a feature or fix a bug
---

## Instructions

1. Write a failing test first. Run it. Confirm it fails.
2. Write the minimum implementation to make it pass.
3. Run the full test suite.
4. Refactor if needed, re-running tests after each change.
5. Never skip the red-green-refactor cycle.
```

Invoke manually with `/tdd` in the TUI, or let Codex activate it automatically when the trigger matches.

### Skills worth building

| Skill | Purpose |
|-------|---------|
| `/tdd` | Enforce red-green-refactor before any implementation |
| `/grill-me` | Challenge your design before writing code, ask hard questions, find edge cases |
| `/pr-prep` | Run linter, type-checker, tests, then draft a PR description |
| `/migration` | Generate database migration from schema diff |
| `/incident` | Structured incident response: gather logs, identify root cause, draft fix |

Skills encode *process*, not knowledge. A `/tdd` skill does not teach Codex what TDD is. It enforces the discipline of writing the test first, running it and iterating. That behavioural constraint is what produces better code.

## Subagents: isolated context for specialised work

Subagents are child agents spawned by a parent session[^5]. Each runs in its own context window with its own system prompt, tools and permissions. The parent orchestrates; the children execute.

### When to use subagents

- **Code review.** A review subagent examines diffs without implementation bias, catching security issues, convention violations and missing tests that the implementing agent overlooked.
- **Parallel exploration.** Three competing approaches investigated simultaneously, each in a separate context, with results compared afterwards.
- **Batch operations.** Refactoring the same pattern across dozens of files, with each subagent handling a subset.

### Defining a subagent

Subagent definitions live in `.codex/agents/` as TOML files[^5]:

```toml
# .codex/agents/security-review.toml
name = "security-review"
description = "Review code changes for security vulnerabilities"
model = "gpt-5.4"
model_reasoning_effort = "high"

[instructions]
content = """
You are a security reviewer. Examine the diff for:
- Injection vulnerabilities (SQL, XSS, command injection)
- Authentication and authorisation gaps
- Secrets or credentials in code
- Unsafe deserialisation
Report findings with severity, location and recommended fix.
"""
```

Subagents consume their own tokens. There is no free parallelism[^5]. Reach for them when the latency or context benefits outweigh the cost.

## Profiles: match the tool to the task

Since v0.134.0, `--profile` is the sole activation path for named permission profiles[^6]. The legacy `[profile.*]` syntax is rejected. Profiles let you switch between configurations without editing files:

```toml
# ~/.codex/config.toml

[permission_profiles.quick]
model = "gpt-5.4-mini"
model_reasoning_effort = "low"
model_verbosity = "low"

[permission_profiles.deep]
model = "gpt-5.5"
model_reasoning_effort = "high"
plan_mode_reasoning_effort = "xhigh"

[permission_profiles.review]
model = "gpt-5.4"
model_reasoning_effort = "high"
```

```bash
# Routine file edits
codex --profile quick "rename all .jsx files to .tsx"

# Complex architecture work
codex --profile deep "design the event sourcing layer"

# Code review
codex --profile review "review the last 3 commits"
```

This is not cosmetic. A quick profile on gpt-5.4-mini with low reasoning burns a fraction of the tokens that a deep session on gpt-5.5 with xhigh reasoning consumes[^7]. Matching profile to task is one of the most effective habits you can build.

## The current model lineup

As of May 2026, these are the models available in Codex CLI[^8]:

| Model | Context | Best for |
|-------|---------|----------|
| gpt-5.5 | 1M | Newest frontier model; state-of-the-art reasoning and coding |
| gpt-5.4 | 1M | Flagship; recommended default for professional work |
| gpt-5.4-mini | 400K | Fast subagent work; 30 per cent of gpt-5.4 quota |
| gpt-5.3-codex | 272K input / 400K total | Coding specialist |
| gpt-5.3-codex-spark | 128K | Near-instant iteration; 1,000+ tokens per second |
| gpt-5.2 | 272K input / 400K total | Previous generation |

Set your default in config.toml:

```toml
model = "gpt-5.5"
```

Or override per session:

```bash
codex -m gpt-5.4-mini "quick formatting fix"
```

## MCP servers: making Codex stack-aware

Model Context Protocol (MCP) servers connect Codex to external systems: GitHub, Linear, Postgres, Figma, documentation sites, internal APIs[^9]. Without MCP, Codex can only read files. With it, Codex becomes aware of your entire stack.

### Configuration

```toml
# .codex/config.toml
[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]

[mcp_servers.postgres]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-postgres"]
env = { DATABASE_URL = "postgresql://localhost:5432/mydb" }
```

### The token cost trap

Every MCP server injects its full tool catalogue into every request. A GitHub MCP server with 93 tools consumes roughly 55,000 tokens per turn[^10]. Before connecting five servers and wondering why your context window fills instantly, audit what you need. If you only use GitHub for PR creation, the built-in shell tools (`gh pr create`) cost a fraction of an MCP server.

**Rule of thumb:** connect MCP servers you use every session. For occasional needs, use shell commands or enable the server only when required.

## Parallel sessions with git worktrees

Running multiple Codex instances across git worktrees is the single biggest productivity unlock available today. Three independent sessions, one implementing, one reviewing, one exploring, operate on isolated copies of your repository without interfering with each other.

```bash
# Create worktrees for parallel work
git worktree add ../myproject-review review-branch
git worktree add ../myproject-explore explore-branch

# Terminal 1: implementation
cd myproject && codex --profile deep

# Terminal 2: code review
cd ../myproject-review && codex --profile review

# Terminal 3: exploration
cd ../myproject-explore && codex --profile quick
```

Each session has its own working directory, git state and context window. The implementing agent never sees the reviewer's findings until you merge. This isolation is the point. It prevents context contamination and lets each agent focus on its specialised task.

## Essential slash commands

The TUI exposes commands that control sessions, context and behaviour without leaving the terminal[^11]:

| Command | What it does |
|---------|-------------|
| `/compact` | Lossy summarisation of conversation history, frees context window capacity |
| `/status` | Shows model, token usage, git branch and sandbox mode |
| `/clear` | Resets session history completely |
| `/permissions` | Inspects the active permission profile |
| `/model <name>` | Switches model mid-session |
| `/plugins` | Lists installed plugins |
| `/undo` | Reverts the last file changes made by Codex |
| `/diff` | Shows pending file changes as a unified diff |

### The compaction discipline

Manual `/compact` at roughly 60 per cent context usage produces better summaries than waiting for automatic compaction at 95 per cent[^12]. After two or three automatic compactions, the model loses track of early decisions. Treat compaction as a deliberate phase transition: finish investigation, compact, then start implementation with a clean context.

## Hooks: event-driven automation

Hooks execute shell commands at specific points in the Codex workflow[^13]:

```bash
# .codex/hooks/post-patch.sh
#!/bin/bash
# Auto-format after every patch Codex applies
npx prettier --write "$CODEX_CHANGED_FILES"
npm run lint:fix
```

Available hook points include pre-exec (before tool execution), post-patch (after file changes) and session lifecycle events. Hooks run outside the model's context. They do not consume tokens and cannot be overridden by the model's reasoning.

## The `codex exec` pipeline

For automation and CI/CD, `codex exec` skips the TUI entirely and returns structured output[^14]:

```bash
# Single-shot execution
codex exec "write unit tests for src/auth.ts" --model gpt-5.4-mini

# Piped workflows
cat failing-tests.txt | codex exec "fix these test failures" --model gpt-5.4

# Structured JSON output
codex exec "list all TODO comments" --output-format json
```

Combined with `--profile quick`, this is the most token-efficient way to run batch operations. Use it in CI pipelines, pre-commit hooks and scripted maintenance tasks.

## Codex doctor

Since v0.135.0, `codex doctor` diagnoses configuration and environment issues before they derail a session[^15]:

```bash
codex doctor
```

It checks API key validity, model access, MCP server connectivity, config.toml syntax and sandbox compatibility. Run it after installation, after changing configuration and whenever something feels wrong. Five seconds of diagnostics saves 20 minutes of debugging.

## Daily habits of effective Codex users

The patterns that separate productive Codex users from frustrated ones are not about prompting:

1. **Edit AGENTS.md multiple times per week.** It is a living document. When Codex makes a mistake, add a rule. When a rule becomes unnecessary because the codebase changed or the linter now catches it, remove it.

2. **Use gpt-5.5 with high or xhigh reasoning for complex work.** Do not default to the cheapest model for everything. The cost difference between gpt-5.4-mini and gpt-5.5 is large, but so is the quality difference on architectural decisions and complex refactors.

3. **Start fresh sessions between phases.** Investigation and implementation have different context needs. Compact or clear between them.

4. **Treat configuration as the primary work.** Writing a good skill or tuning a profile takes 30 minutes and pays off across hundreds of sessions. Writing a better prompt takes 30 seconds and pays off once.

5. **Run `codex doctor` after any config change.** Catch problems before they cost you tokens and time.

6. **Audit MCP servers quarterly.** Remove servers you have not used in a month. Each idle server burns tokens silently on every turn.

## The mastery progression

| Level | Focus | Key investment |
|-------|-------|----------------|
| **Beginner** | Typing prompts, waiting for output | None |
| **Intermediate** | AGENTS.md, profiles, `/compact` discipline | Two hours of config work |
| **Advanced** | Skills, subagents, MCP servers, worktree parallelism | One to two days building infrastructure |
| **Expert** | Hooks, `codex exec` pipelines, CI integration, cross-agent orchestration | Ongoing refinement |

Most developers plateau at intermediate. The jump to advanced requires investing time in infrastructure rather than prompts, and accepting that the 30 minutes spent writing a skill saves hours across future sessions.

## Conclusion

Codex CLI becomes powerful through infrastructure investment, not superior prompting. The prompt is the last mile. Everything before it, AGENTS.md, skills, subagents, profiles, MCP servers, hooks, worktree isolation, determines whether that last mile produces throwaway suggestions or production-ready code.

The developers who get the most from Codex CLI are not writing better prompts. They are building better environments for the agent to operate in. Start with AGENTS.md. Add a verification command. Create one skill for your most common task. The compound returns begin immediately.

## Citations

[^1]: [AGENTS.md Advanced Patterns — Codex Resources](https://developers.openai.com/codex/guides/agents-md)

[^2]: [Boris Cherny — CLAUDE.md design philosophy, adapted for AGENTS.md](https://www.cherny.dev)

[^3]: [Codex CLI Configuration Reference — OpenAI Developers](https://developers.openai.com/codex/config-reference)

[^4]: [The Codex CLI Skills Ecosystem: agentskills.io and Community Skills](https://agentskills.io)

[^5]: [Codex CLI Subagents: TOML Format, Parallelism and spawn_agents_on_csv — Codex Resources](https://danielvaughan.github.io/codex-resources/articles/2026-03-26-codex-cli-subagents-toml-parallelism/)

[^6]: [Codex CLI v0.134.0 Release Notes — GitHub](https://github.com/openai/codex/releases)

[^7]: [Codex CLI Performance Optimisation: Token Overhead, Hidden Costs and Tuning Tactics — Codex Resources](https://danielvaughan.github.io/codex-resources/articles/2026-04-08-codex-cli-performance-optimization/)

[^8]: [Codex Models — OpenAI Developers](https://developers.openai.com/codex/models)

[^9]: [Codex CLI MCP Integration — Codex Resources](https://danielvaughan.github.io/codex-resources/articles/2026-03-26-codex-cli-mcp-integration/)

[^10]: [MCP Token Trap: Why Your AI Agent Burns 35x More Tokens Than a CLI — OnlyCLI](https://onlycli.github.io/OnlyCLI/blog/mcp-token-cost-benchmark/)

[^11]: [Codex CLI TUI Shortcuts and Slash Commands: The Complete Reference — Codex Resources](https://danielvaughan.github.io/codex-resources/articles/2026-04-08-codex-cli-tui-shortcuts-slash-commands/)

[^12]: [Why Is My Codex CLI Token Usage Suddenly So High? — BSWEN](https://docs.bswen.com/blog/2026-03-02-codex-cli-token-usage-spike/)

[^13]: [Codex CLI Hooks: Complete Guide to Events, Policy and Patterns — Codex Resources](https://danielvaughan.github.io/codex-resources/articles/2026-04-15-codex-cli-hooks-complete-guide-events-policy-patterns/)

[^14]: [codex exec: Unix Pipeline Integration and Structured Output — Codex Resources](https://danielvaughan.github.io/codex-resources/articles/2026-04-01-codex-exec-unix-pipeline-structured-output/)

[^15]: [Codex CLI v0.135.0 Release Notes — GitHub](https://github.com/openai/codex/releases/tag/v0.135.0)
