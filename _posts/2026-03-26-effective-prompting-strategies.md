---
title: "Effective Prompting Strategies for Codex CLI"
date: 2026-03-26
tags: [prompting, context, AGENTS.md, skills, workflow]
---
![Sketchnote: Effective Prompting Strategies for Codex CLI](/sketchnotes/articles/2026-03-26-effective-prompting-strategies.png)

*Based on official OpenAI documentation, community discussion, and developer best practices. Published 2026-03-26.*

---

## The Core Principle

Codex is not a one-shot assistant. The most effective users treat it as a **configurable teammate**: invest upfront in context files and prompt structure, then reuse that investment across every task.

---

## 1. Prompt Anatomy: Four Essential Elements

Every Codex prompt works best with these four components:

```
Goal       — what "done" looks like in one concrete sentence
Context    — environment, framework, language version, existing patterns
Constraints — what to avoid (no new deps, don't touch tests, match existing naming)
Completion — how to know it's finished ("run the tests, they should pass")
```

**Example:**

```
Goal: Add pagination to the /api/posts endpoint
Context: Express.js, PostgreSQL, follows existing /api/users pattern
Constraints: Don't modify the database schema, don't add new npm packages
Completion: GET /api/posts?page=2&limit=20 returns correct results, existing tests still pass
```

One specific example in the prompt is worth more than a paragraph of explanation.

---

## 2. AGENTS.md: Move Durable Context Out of Prompts

Every time you copy-paste the same background into a prompt, it belongs in AGENTS.md instead.

**What to put in AGENTS.md:**

```markdown
# Project: Payment Service

## Build commands
npm run build && npm test

## Test command
npm run test:unit -- --coverage

## Conventions
- Async/await everywhere — no .then() chains
- Error messages in sentence case, no trailing periods
- All amounts in pence (integers), not pounds (floats)

## Do not modify
- src/legacy/chargebee.ts (deprecated, read-only for reference)
- Any file ending in .generated.ts
```

**Rule:** A short, accurate AGENTS.md beats a long vague one. If a rule is vague enough that Codex might not know what to do with it, remove it.

AGENTS.md hierarchy (later overrides earlier):

```
~/.codex/AGENTS.md          # personal global defaults
.codex/AGENTS.md            # repo root
src/payments/AGENTS.md      # subdirectory-specific rules
```

Bootstrap with `codex /init` to scaffold from your repo.

---

## 3. Custom System Prompts for Your Stack

Beyond AGENTS.md, a per-project system prompt sharpens Codex's defaults without polluting the repo.

In `~/.codex/config.toml`:

```toml
[profiles.swift]
system_prompt = "You are working on a Swift/ObjC iOS app. Use Swift concurrency (async/await). Prefer SwiftUI for new views. Match existing code style exactly."

[profiles.bugfix]
system_prompt = "You are a meticulous debugger. Before suggesting a fix, articulate the root cause. Include a regression test with every fix. No speculative changes."
```

Usage: `codex --profile swift "fix the memory leak in ImageCache"`

---

## 4. Plan Mode Before Complex Tasks

For anything ambiguous or multi-step, use `/plan` (or `Shift+Tab`) before diving in:

```
/plan Propose a migration strategy for moving our auth from sessions to JWT
```

Plan mode lets Codex gather context and ask clarifying questions first. The output is a reviewable plan you can edit before execution begins.

**When to use `/plan`:**

- Cross-cutting refactors (touching 10+ files)
- Architecture changes (adding a new service, changing a data model)
- Debugging hard-to-reproduce issues
- Anything where the approach matters as much as the output

**When to skip `/plan`:**

- Well-scoped tasks with obvious implementation ("add a unit test for this function")
- Quick fixes with clear root cause
- Chores with established patterns (bumping dependencies, updating docs)

---

## 5. Verify-First Prompting

Tell Codex to validate its own work. Don't leave the success criteria implicit:

```
Refactor the order processing module to remove the circular dependency.
Run the tests after each change. Only proceed if tests stay green.
When complete, run `npm run typecheck` and confirm 0 errors.
```

This pattern ("do X, verify Y, only continue if Z") dramatically reduces the chance of silent failures.

---

## 6. Checkpoint Pattern for Long Tasks

For long-running tasks, add intermediate checkpoints:

```
Migrate the legacy authentication module to the new auth service.

After each step, pause and confirm:
1. Tests still pass
2. No TypeScript errors
3. Git state is clean (committed)

Steps:
1. Extract the token validation logic to src/auth/validate.ts
2. Update the middleware to use the new module
3. Remove the legacy auth.js file
4. Update all imports
```

This prevents Codex from making all changes at once and making it harder to debug partial failures.

---

## 7. Reasoning Effort Selection

Match effort to task complexity:

| Level | Flag | Use for |
|-------|------|---------|
| `low` | `-e low` | Quick edits, obvious bugs, rename operations |
| `medium` (default) | `-e medium` | Most interactive coding — good balance |
| `high` | `-e high` | Complex algorithms, cross-cutting refactors |
| `xhigh` | `-e xhigh` | Hardest reasoning tasks: architecture, security review |

**Note:** Plan mode forces medium reasoning. Long-running subagent tasks often benefit from high effort on the parent, low effort on worker subagents.

---

## 8. Parallelism Signals

Tell Codex explicitly when work is parallelisable. The model can use parallel tool calls but needs permission to assume tasks are independent:

```
Write unit tests for the following three modules in parallel:
- src/auth/validate.ts
- src/payments/process.ts
- src/notifications/sender.ts

Each test file should use the existing test helpers in __tests__/helpers.ts.
```

This unlocks `multi_tool_use.parallel` behaviour — without the signal, Codex may sequence work unnecessarily.

---

## 9. Slash Commands for Session Control

| Command | Use |
|---------|-----|
| `/plan [prompt]` | Switch to plan mode, optionally with an inline prompt |
| `/model` | Switch model or reasoning level mid-session |
| `/compact` | Summarise earlier context to free tokens |
| `/fork` | Branch the current session for a parallel investigation |
| `/review` | Code-review the current diff against base branch |
| `/personality` | Adjust communication style without rewriting prompts |

---

## 10. PRD-First Prompting (Advanced)

For complex new features, write the requirements before writing the prompt:

1. Describe your feature idea to **Claude or ChatGPT** in rough form
2. Ask: *"Turn this into a structured PRD with acceptance criteria and technical constraints"*
3. Feed the PRD to Codex as the opening prompt

This works because Codex performs better with well-structured, complete context. You spend your Codex usage on implementation, not clarification.

---

## Common Prompting Pitfalls

| ❌ Anti-pattern | ✅ Fix |
|----------------|--------|
| Vague goals ("improve this function") | Specific outcomes ("make this function handle null inputs without throwing") |
| Repeating the same context in every prompt | Move it to AGENTS.md |
| No completion criteria | Add a test/verify step to every prompt |
| Watching Codex work step by step | Use plan mode first, then let it run |
| Single massive prompt for a complex change | Use checkpoint pattern or subagents |
| Not telling Codex about safe-to-parallelize tasks | Explicitly list independent tasks |

---

## Sources

- [Best practices — Codex Developers](https://developers.openai.com/codex/learn/best-practices)
- [Custom Prompts — Codex Developers](https://developers.openai.com/codex/custom-prompts)
- [Slash commands — Codex Developers](https://developers.openai.com/codex/cli/slash-commands)
- [Codex Prompting Guide — OpenAI Cookbook](https://developers.openai.com/cookbook/examples/gpt-5/codex_prompting_guide)
- [GitHub Discussion: Custom system prompts — openai/codex #7296](https://github.com/openai/codex/discussions/7296)
