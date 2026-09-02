---
title: "Codex CLI Prompt Library: 20 Battle-Tested Patterns for Code Review, Refactoring, Testing, and Documentation"
type: Technical Article
timestamp: 2026-06-04T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-06-04-codex-cli-prompt-library-battle-tested-patterns-code-review-refactoring-testing-documentation"
tags: ["codex-cli", "prompts", "prompt-engineering", "code-review", "refactoring", "testing", "documentation", "workflows", "productivity", "v0.136"]
date: 2026-06-04T09:00:00+00:00
last_modified_at: 2026-09-02T08:17:51+01:00
---
# Codex CLI Prompt Library: 20 Battle-Tested Patterns for Code Review, Refactoring, Testing, and Documentation


---

Prompting principles are well documented. Concrete prompts are not. Senior developers waste tokens discovering what works through trial and error, when the same patterns surface repeatedly across production teams. This article provides twenty copy-paste prompts organised by workflow, each following the outcome-first structure that GPT-5.5 and GPT-5.3-Codex respond to best: **goal**, **context**, **constraints**, and **done-when** criteria[^1][^2].

Every prompt assumes Codex CLI v0.136 with a properly configured `AGENTS.md` at the repository root[^3]. Adapt file paths and tool names to your stack.

## The Prompt Structure That Works

Before the library, a brief anatomy. The most effective Codex CLI prompts share four components[^1]:

```mermaid
flowchart LR
    A[Goal] --> B[Context]
    B --> C[Constraints]
    C --> D[Done when]
    D --> E[Agent executes]
```

- **Goal** — the observable outcome, not the steps to reach it.
- **Context** — files, directories, error output, or screenshots the agent needs.
- **Constraints** — architectural rules, forbidden patterns, and style requirements.
- **Done when** — the verifiable end state that tells the agent to stop.

Shorter prompts with clear outcomes outperform verbose step-by-step instructions[^2]. The agent chooses its own execution path; your job is to define what "finished" looks like.

## Code Review Prompts

### 1. Targeted Diff Review

```text
Review the current git diff. Focus on:
- Race conditions and shared mutable state
- Error paths that swallow exceptions
- Public API changes that break backward compatibility
Done when: each finding has a file path, line range, severity (critical/warning/info), and a one-sentence fix suggestion.
```

Use after staging changes, before committing. For a broader review of the entire working tree, use `/review` instead[^4].

### 2. Security-Focused Review

```text
Review @src/ for security vulnerabilities. Check for:
- SQL injection, XSS, and path traversal
- Hardcoded secrets or credentials
- Insecure deserialization
- Missing input validation on API boundaries
Constraints: follow OWASP Top 10 2025 categories. Do not modify any files.
Done when: a markdown table of findings with CWE IDs, affected files, and remediation steps.
```

### 3. Performance Review

```text
Read @src/api/ and identify the three costliest operations by likely wall-clock time. For each, explain why it is slow and propose a fix that does not change the public interface.
Constraints: assume PostgreSQL 16 as the database. Prefer query-level fixes over caching.
Done when: each finding includes a before/after explanation and estimated impact.
```

## Refactoring Prompts

### 4. Extract Module

```text
The authentication logic in @src/server.ts is tangled with request handling. Extract it into a new @src/auth/ module with:
- A clean public interface (types + functions)
- No circular imports
- All existing tests still passing
Constraints: do not change any HTTP response shapes. Keep the diff minimal.
```

### 5. Reduce Complexity

```text
Refactor @src/utils/parser.ts to reduce cyclomatic complexity below 10 per function. Preserve identical behaviour — run the existing test suite before and after to confirm.
Constraints: no new dependencies. British English in comments.
Done when: all functions score below 10 and tests pass.
```

### 6. Dead Code Removal

```text
Find and remove all unreachable code, unused exports, and dead imports in @src/. Use rg to confirm each removal has zero references before deleting.
Constraints: do not remove anything prefixed with _ (internal convention for future use). Run tests after each batch of removals.
Done when: a summary listing every file changed and what was removed.
```

### 7. Type Strictness Migration

```text
Enable strict: true in tsconfig.json and fix all resulting type errors in @src/. Prefer narrowing and type guards over type assertions. Do not use 'any' or 'as unknown as T' patterns.
Constraints: fix files in dependency order (leaf modules first). Run tsc --noEmit after each file.
Done when: tsc --noEmit exits 0 with strict: true and no suppression comments were added.
```

## Testing Prompts

### 8. Coverage Gap Filler

```text
Run the test suite with coverage reporting. Identify the three files with the lowest branch coverage that contain business logic (not config or types). Write tests to bring each above 80% branch coverage.
Constraints: follow the existing test style in @tests/. Use the same assertion library. No mocking of the module under test's internals.
Done when: coverage report shows all three files above 80% branch coverage.
```

### 9. Edge Case Generator

```text
Read @src/pricing/calculator.ts and generate edge-case unit tests covering:
- Zero and negative inputs
- Boundary values at tier thresholds
- Currency rounding edge cases
- Concurrent access patterns (if applicable)
Constraints: one test file, descriptive test names, no snapshot tests.
Done when: all new tests pass and cover at least 5 edge cases not in the existing suite.
```

### 10. Integration Test Scaffold

```text
Create an integration test for the /api/orders endpoint that:
1. Seeds a test database with known state
2. Makes HTTP requests through the actual router
3. Asserts response shapes, status codes, and database side effects
4. Cleans up after itself
Constraints: use the test utilities in @tests/helpers/. Do not mock the database layer. Each test must be independently runnable.
```

### 11. Flaky Test Diagnosis

```text
The test "should sync user preferences" in @tests/sync.test.ts fails intermittently in CI. Diagnose the flakiness:
1. Read the test and its dependencies
2. Identify timing assumptions, shared state, or network dependencies
3. Propose a fix that makes the test deterministic
Constraints: do not increase test timeouts as a fix. Prefer dependency injection over mocking.
Done when: the fix is applied and the test passes 10 consecutive runs.
```

## Documentation Prompts

### 12. API Documentation from Source

```text
Read every exported function in @src/sdk/client.ts and generate JSDoc comments for each. Include:
- A one-line summary
- @param descriptions with types
- @returns description
- @throws for known error cases
- @example with a minimal usage snippet
Constraints: do not change function signatures. Use British English.
Done when: tsc compiles without errors and every export has complete JSDoc.
```

### 13. Architecture Decision Record

```text
We are switching from REST to gRPC for internal service communication. Write an ADR (Architecture Decision Record) documenting:
- Context: why the current REST approach is insufficient
- Decision: gRPC with Protocol Buffers
- Consequences: what changes, what breaks, what improves
- Migration plan: phased approach with rollback strategy
Save to @docs/adr/005-grpc-migration.md following the format in existing ADRs.
```

### 14. README Refresh

```text
Update @README.md to reflect the current state of the project. Read the package.json, directory structure, and AGENTS.md to understand what has changed. Ensure:
- Installation instructions work (test them)
- All example commands are current
- Environment variable documentation matches .env.example
- Links are valid
Constraints: keep the existing structure. Do not add badges or boilerplate.
Done when: every code block in the README executes without error.
```

## Bug-Fixing Prompts

### 15. Reproduction-First Bug Fix

```text
Bug: users report that CSV export truncates rows beyond 10,000 records.
Repro steps:
1. Seed the database with 15,000 records: npm run seed -- --count=15000
2. Trigger export: curl -X POST localhost:3000/api/export
3. Count rows in the output file
Constraints: do not change the export API contract. Fix must handle arbitrarily large datasets without loading everything into memory.
Done when: export produces all 15,000 rows and a regression test covers this case.
```

### 16. Stack Trace Investigation

```text
This error appears in production logs:

TypeError: Cannot read properties of undefined (reading 'email')
    at UserService.notify (/app/src/services/user.ts:142:38)
    at OrderProcessor.complete (/app/src/processors/order.ts:87:22)

Read the referenced files, trace the data flow, identify the root cause, and fix it. Add a guard or validation at the appropriate layer.
Done when: the fix handles the undefined case gracefully, a test covers it, and the existing suite still passes.
```

## Exploration and Understanding Prompts

### 17. Dependency Impact Analysis

```text
We need to upgrade lodash from 4.x to 5.x. Before making any changes:
1. Find every import of lodash across the codebase
2. Check the lodash 5 migration guide for breaking changes
3. List which of our usages are affected
4. Estimate effort (trivial/moderate/significant) for each change
Output a markdown table. Do not make any code changes yet.
```

### 18. Codebase Onboarding Summary

```text
I am new to this codebase. Read the top-level directory structure, AGENTS.md, package.json (or equivalent), and the main entry point. Provide:
- A one-paragraph summary of what this project does
- The tech stack with versions
- How to run it locally (with exact commands)
- The three most important files to read first
- Known gotchas or surprising patterns
Constraints: be concise. Target a senior developer who has never seen this repo.
```

## CI/CD and Automation Prompts

### 19. Pre-Commit Quality Gate (codex exec)

```bash
codex exec --sandbox read-only \
  "Review the staged changes (git diff --cached) for:
   - Type errors and lint violations
   - Security issues (hardcoded secrets, injection vectors)
   - Test coverage gaps in modified code
   Output a JSON object with fields: passed (boolean), findings (array), summary (string)" \
  --output-schema quality-gate.json
```

This pattern integrates directly into a `pre-commit` hook or CI step. The `--output-schema` flag ensures machine-readable output for downstream tooling[^5][^6].

### 20. Automated Changelog Entry (codex exec)

```bash
codex exec \
  "Read the git log since the last tag (git describe --tags --abbrev=0).
   Categorise each commit as: feature, fix, refactor, docs, or chore.
   Write a changelog entry in Keep a Changelog format.
   Save to CHANGELOG.md, prepending to the existing content.
   Done when: CHANGELOG.md has a new Unreleased section with all commits categorised." \
  --sandbox workspace-write
```

## Composing Prompts with Skills

For prompts you use daily, convert them to skills rather than retyping them[^7]. A `SKILL.md` wrapping prompt #8 (coverage gap filler) looks like this:

```markdown
---
name: coverage-gap-filler
description: Find and fill the three lowest-coverage business logic files
---

Run the test suite with coverage. Identify the three files with the lowest
branch coverage containing business logic. Write tests to bring each above
80%. Follow the existing test style. Report results as a markdown table.
```

Place it in `.codex/skills/coverage-gap-filler/SKILL.md` and invoke with `$coverage-gap-filler` from the TUI[^7].

## Anti-Patterns to Avoid

Three prompting mistakes waste the most tokens in practice[^2]:

1. **Step-by-step instructions** — "First read file X, then open file Y, then..." forces a rigid execution path. State the outcome and let the agent plan.
2. **Vague goals without done-when criteria** — "Make the code better" produces unfocused changes. Specify what "better" means: "Reduce response time below 200ms at p99."
3. **Overloading a single prompt** — cramming five unrelated tasks into one prompt degrades quality. Use one prompt per coherent unit of work, or spawn subagents for parallel tasks[^8].

## Citations

[^1]: OpenAI, "Best practices – Codex," OpenAI Developers, 2026. [https://developers.openai.com/codex/learn/best-practices](https://developers.openai.com/codex/learn/best-practices)

[^2]: OpenAI, "Prompting – Codex," OpenAI Developers, 2026. [https://developers.openai.com/codex/prompting](https://developers.openai.com/codex/prompting)

[^3]: OpenAI, "Custom instructions with AGENTS.md – Codex," OpenAI Developers, 2026. [https://developers.openai.com/codex/guides/agents-md](https://developers.openai.com/codex/guides/agents-md)

[^4]: OpenAI, "Workflows – Codex," OpenAI Developers, 2026. [https://developers.openai.com/codex/workflows](https://developers.openai.com/codex/workflows)

[^5]: OpenAI, "Non-interactive mode – Codex," OpenAI Developers, 2026. [https://developers.openai.com/codex/noninteractive](https://developers.openai.com/codex/noninteractive)

[^6]: OpenAI, "Command line options – Codex CLI," OpenAI Developers, 2026. [https://developers.openai.com/codex/cli/reference](https://developers.openai.com/codex/cli/reference)

[^7]: OpenAI, "Agent Skills – Codex," OpenAI Developers, 2026. [https://developers.openai.com/codex/skills](https://developers.openai.com/codex/skills)

[^8]: OpenAI, "Codex Prompting Guide," OpenAI Cookbook, 2026. [https://developers.openai.com/cookbook/examples/gpt-5/codex_prompting_guide](https://developers.openai.com/cookbook/examples/gpt-5/codex_prompting_guide)
