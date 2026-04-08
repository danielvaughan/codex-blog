---
title: "Test-First Development with Codex: Using TDD as the Agent Feedback Loop"
date: 2026-03-28
tags: [tdd, testing, agentic, feedback-loop, agents-md, best-practices]
summary: "How TDD transforms agentic workflows: tests give Codex a verifiable goal and automatic feedback without human intervention. Writing tests before dispatching Codex; integrating pytest/Jest/Vitest verification loops in AGENTS.md; how test coverage doubles as agent progress tracking."
---

![Sketchnote diagram for: Test-First Development with Codex: Using TDD as the Agent Feedback Loop](/sketchnotes/articles/2026-03-28-test-first-development-codex-tdd-feedback-loop.png)



# Test-First Development with Codex: Using TDD as the Agent Feedback Loop

The single biggest problem with autonomous agents is knowing when they're done. A human developer can feel when code "feels right." An agent cannot. But a failing test suite can — and that's the key insight behind using TDD as your primary Codex feedback mechanism.

Write the tests first. Let the tests tell the agent when it has succeeded.

---

## Why Tests Are the Perfect Agent Interface

When you dispatch Codex without tests, you're asking it to work toward a subjective goal: "make this feature work." The agent has no reliable way to verify its own progress. It may produce code that looks correct but fails edge cases, introduces regressions, or satisfies the literal description while missing the intent.

When you write tests first, you change the nature of the task entirely:

- **The goal becomes objective** — make these tests pass
- **Progress is measurable** — 3/10 tests passing → 7/10 → 10/10
- **Regression detection is automatic** — if an earlier test starts failing, Codex knows immediately
- **Human review becomes faster** — green CI is a form of proof, not just documentation

OpenAI's own engineering team uses this pattern at scale. Their internal finding: routing 1,500 PRs through Codex with TDD-anchored feedback loops produced 3.5 PRs per engineer per day — rising as the team scaled, not falling.

---

## The TDD-First Dispatch Pattern

The canonical workflow:

```
1. Write failing tests → commit to branch
2. Dispatch Codex: "make these tests pass"
3. Codex implements → runs tests → iterates until green
4. Human reviews diff → reviews test quality → merges
```

Compare this to the naive pattern:

```
1. Dispatch Codex: "implement feature X"
2. Codex produces code
3. Human must understand and verify everything
4. Human writes tests (often skipped)
```

In the TDD pattern, **the tests encode your intent** so you don't have to re-articulate it during review. You wrote the spec up front; Codex executed it.

---

## Configuring TDD in AGENTS.md

Add test instructions directly to your repo's `AGENTS.md`:

```markdown
## Testing Protocol

- Always run tests before declaring a task complete
- Command: `pytest tests/ -v` (Python) / `npm test` (Node) / `cargo test` (Rust)
- If tests fail, fix the implementation — do not modify the tests unless the test itself has a bug
- Report: number of tests passing / total before and after your changes
- Never mark a task as done if any test is RED

## TDD Convention

When a task includes a `tests/` directory or `*.test.*` files, assume test-first:
1. Read the failing tests to understand the requirement
2. Implement the minimal code to make them pass
3. Refactor only after tests are green
```

The critical rule: **"do not modify tests unless the test has a bug"**. Without this, agents will make tests pass by weakening them. Codex respects explicit rules in `AGENTS.md`.

---

## PostToolUse Hook: Auto-Test on Every Write

Codex v0.117.0 introduced `PostToolUse` hooks that fire after Bash tool execution. Combined with `PreToolUse`, you can build a continuous test loop that runs automatically after every file write:

```toml
# ~/.codex/config.toml
[[hooks]]
trigger = "PostToolUse"
matcher = { tool = "bash" }
run = ["bash", "-c", "if echo '$CODEX_TOOL_OUTPUT' | grep -q '.py'; then pytest tests/ -q --tb=short 2>&1 | tail -20; fi"]
```

A simpler approach — add a test-runner step to the `AGENTS.md` workflow:

```markdown
## After Every File Change

Run: `pytest --tb=short -q` (or equivalent for your stack)
If any test fails: fix before proceeding to the next step.
```

This converts Codex into a continuous verification loop without any human checkpoints.

---

## Test Coverage as Agent Progress Tracking

When you combine test-first development with coverage reporting, the coverage report becomes a real-time progress dashboard for long-running Codex tasks.

```markdown
## In AGENTS.md

After completing implementation:
Run: `pytest --cov=src --cov-report=term-missing -q`
Target: ≥90% line coverage before flagging task complete.
Report the final coverage percentage as the last message in the session.
```

This gives you two review signals when the task finishes:
1. **Tests green** — behaviour is correct
2. **Coverage ≥ 90%** — behaviour is tested comprehensively

Both are automatic, auditable, and require no additional human effort to produce.

---

## Framework: Per-Language Test Integration

### Python (pytest)

```markdown
## AGENTS.md — Python

Test runner: `pytest tests/ -v --tb=short`
Coverage: `pytest --cov=src --cov-report=term-missing -q`
Watch mode (for long sessions): `ptw -- -q`
Minimum coverage: 85%
```

### JavaScript / TypeScript (Vitest or Jest)

```markdown
## AGENTS.md — Node

Test runner: `npm test -- --run` (Vitest) / `npx jest --forceExit`
Coverage: `npm test -- --coverage`
Watch mode: `npm test` (Vitest auto-watches)
Threshold in vitest.config.ts: 80% lines/branches
```

### Rust

```markdown
## AGENTS.md — Rust

Test runner: `cargo test`
Coverage: `cargo tarpaulin --out Stdout`
Clippy: `cargo clippy -- -D warnings` (treat warnings as errors)
```

---

## shinpr/agentic-code: A TDD Framework for Codex

The open-source [shinpr/agentic-code](https://github.com/shinpr/agentic-code) framework (added to `ecosystem-tools.md`) codifies TDD-first workflows for Codex and other agents. Key principles:

1. **Requirements analysis first** — agent reads spec, identifies ambiguities, confirms understanding before writing code
2. **Architecture planning** — brief design document before any implementation
3. **Test-first development** — tests written from the spec before implementation
4. **Quality gate** — review only after coverage thresholds are met
5. **Fresh-session review** — the reviewing agent must be a **different session** (same assumptions carry over within one context window, making self-review unreliable)

The fresh-session review insight is particularly important: asking Codex to review its own work in the same thread is like asking someone to proofread their own writing immediately after writing it. A separate session, or a different model entirely, catches what the first pass missed.

---

## Metaswarm: TDD Enforcement at Scale

For teams running Codex in multi-agent orchestration, [dsifry/metaswarm](https://github.com/dsifry/metaswarm) takes TDD enforcement to its logical conclusion:

- Coverage thresholds in `.coverage-thresholds.json` act as **blocking gates** before PR creation — not suggestions, gates
- Pre-push hooks (Husky) run lint + typecheck + coverage on every `git push`
- The executing agent and the reviewing agent are **always different models** (cross-model adversarial review)
- 9-phase workflow with 6 concurrent specialist reviewers at the design gate

This is production-proven: extracted from a codebase that has run 100% TDD coverage across hundreds of PRs with no human intervention.

---

## The Compound Benefit: Tests as Documentation

Well-written tests are the best form of documentation for future Codex sessions. When you start a new task:

1. Existing tests tell Codex what the current behaviour *is*
2. New failing tests tell Codex what the new behaviour *should be*
3. Codex can explore the test suite to understand conventions, edge cases, and intent

This compounds over time. The more comprehensive your tests, the less you need to explain in `AGENTS.md` or per-task prompts. The tests carry the context.

---

## Quick-Start Checklist

Before dispatching Codex on a new task:

- [ ] Write 2–5 failing tests that define the expected behaviour
- [ ] Commit the tests: `git commit -m "test: failing tests for [feature]"`
- [ ] Set `AGENTS.md` rule: "do not modify tests unless the test itself has a bug"
- [ ] Add post-run coverage report to AGENTS.md
- [ ] Consider a fresh-session code review after Codex completes

The test suite is your contract with the agent. Write it clearly and Codex will do the rest.

---

## See Also

- [AGENTS.md Advanced Patterns](/codex-resources/articles/2026-03-26-agents-md-advanced-patterns/)
- [Codex CLI Hooks Deep Dive](/codex-resources/articles/2026-03-26-codex-cli-hooks-deep-dive/)
- [Staying Engaged with Your Codebase in an Agentic World](/codex-resources/articles/2026-03-27-staying-engaged-codebase-agentic-world/)
- [The Proof of Work Principle](/codex-resources/articles/2026-03-27-proof-of-work-principle-agents/)
- [shinpr/agentic-code](https://github.com/shinpr/agentic-code) — TDD framework for Codex
- [dsifry/metaswarm](https://github.com/dsifry/metaswarm) — production multi-agent TDD enforcement

*Published: 2026-03-28 · Written by Andy*
