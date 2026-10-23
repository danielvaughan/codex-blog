---
title: "Subagents with Verifiable Goals: Moving from Vague Instructions to Auditable Completion"
date: 2026-10-21T15:00:00+00:00
last_modified_at: 2026-10-23T23:04:40+01:00
parent: "Articles"
nav_order: 1199
tags:
  - codex-cli
  - subagents
  - agents
  - verifiable-goals
  - agents-md
  - hooks
  - aiewf-2026
  - command-centre
  - auditing
copy_edited: true
---

# Subagents with Verifiable Goals: Moving from Vague Instructions to Auditable Completion

![Sketchnote diagram for: Subagents with Verifiable Goals: Moving from Vague Instructions to Auditable Completion](/sketchnotes/articles/2026-10-21-subagents-with-verifiable-goals-vague-instructions-auditable-completion-codex-cli-aiewf-2026.png)

**Source:** Charlie Guo & Gabriel Chua, "Cooking with Codex" workshop, AI Engineer World's Fair 2026 (youtube.com/watch?v=FwiS44-xyYw, 2026-10-06)
**Impact:** Teams using Codex CLI multi-agent workflows; developers deploying subagents for overnight or scheduled work; organisations with audit or compliance requirements on AI-assisted code changes

---

There is a meaningful difference between telling a subagent "write the tests" and telling a subagent "write the tests such that `npm test -- --passWithNoTests` exits 0 and coverage for `src/auth/` is at least 80%."

The first instruction is a vague brief. The subagent will produce something, and you will discover whether it constitutes success when you review the output. The second instruction is a verifiable goal: the subagent knows what success looks like before it starts, can check its own output against the criterion before reporting completion, and can surface a structured result — pass or fail — rather than a pile of files for a human to assess.

Charlie Guo and Gabriel Chua demonstrate this distinction throughout their AIEWF 2026 workshop, using Codex CLI's Command Centre subagent view as the substrate.[^1] The pattern they show is not specific to any particular task type: it applies to any subagent dispatch where the coordinator needs to know whether the subagent finished correctly, not just whether it finished.

---

## What makes a goal verifiable

A verifiable goal has two properties: it can be expressed as a machine-executable check, and the check is unambiguous — it produces a pass or fail result, not a judgment call.

"Write the tests" is not verifiable. There is no shell command that can tell you whether the tests are good.

"Write the tests such that `npm test` exits 0" is verifiable but incomplete. A subagent that deletes the existing tests and adds a single `it('passes', () => {})` would satisfy this criterion.

"Write the tests such that `npm test` exits 0 and `jest --coverage` reports at least 80% line coverage for `src/auth/`" is verifiable and harder to game. The subagent can confirm this with two commands before reporting completion.

The goal of writing verifiable criteria is not to anticipate every possible failure mode. It is to close the gap between "the subagent stopped producing output" and "the subagent completed the task correctly." Most of that gap can be closed with two or three well-chosen checks.

---

## Visible progress in the Command Centre

The Codex CLI Command Centre subagent view shows each dispatched subagent's status in real time: which step the subagent is currently executing, what tool calls it has made, and whether it has completed or is waiting.[^2]

This visibility is more useful when subagents have verifiable goals, because the progress view then reflects meaningful stages — "running verification check," "coverage check passed," "reporting completion to coordinator" — rather than an undifferentiated stream of tool calls.

Guo and Chua note that visible progress tracking changes the experience of supervising a multi-agent run. Without it, you wait and then inspect the final output. With it, you can see whether a subagent is making progress, has stalled, or has reached a checkpoint that requires your input — and you can intervene at the right moment rather than after the fact.

The design implication: structure your AGENTS.md task blocks so that each subagent's verification steps are visible as named stages, not embedded silently in the completion logic. If the Command Centre surfaces tool call names, naming your verification scripts descriptively — `verify-auth-coverage.sh` rather than `check.sh` — makes the progress view informative.

---

## Three patterns for completion criteria

Guo and Chua identify three practical patterns for expressing verifiable completion criteria in subagent work.

### Pattern 1: Command exit code

The simplest criterion. The subagent's task is complete when a specified command exits 0.

```markdown
**Completion criterion:** `cargo test --lib 2>&1 | grep -q "test result: ok"` exits 0.
```

Advantages: no additional tooling required; runs instantly; unambiguous. Limitation: susceptible to a subagent that satisfies the exit code criterion without doing the underlying work (deleting tests, trivially passing assertions). Use exit-code criteria for tasks where gaming the criterion would be obviously detectable on human review.

### Pattern 2: File existence and content assertion

The subagent's task is complete when a specified file exists, is non-empty, and contains specified content.

```markdown
**Completion criterion:** `docs/ADR-042-auth-strategy.md` exists, is at least 400 words, and contains the sections "Context", "Decision", and "Consequences".
```

A verification script for this criterion:

```bash
#!/bin/bash
FILE="docs/ADR-042-auth-strategy.md"

[ -f "$FILE" ] || { echo "FAIL: file missing"; exit 1; }

word_count=$(wc -w < "$FILE")
[ "$word_count" -ge 400 ] || { echo "FAIL: $word_count words (need 400+)"; exit 1; }

for section in "Context" "Decision" "Consequences"; do
  grep -q "## $section" "$FILE" || { echo "FAIL: missing section '$section'"; exit 1; }
done

echo "PASS: ADR-042 verified ($word_count words, required sections present)"
exit 0
```

Advantages: tests the output directly; harder to game than exit-code alone; human-readable failure messages. Use for documentation, configuration, and structured output tasks.

### Pattern 3: AGENTS.md assertion block

For tasks that are part of a longer pipeline, express the completion criterion as a named assertion in AGENTS.md that both the subagent and the coordinator can reference.

```markdown
## Subagent: auth-test-writer

**Brief:** Write unit tests for `src/auth/`. Focus on `src/auth/token.ts` and `src/auth/session.ts`.

**Completion assertions:**
- `assert:tests-pass`: `npm test -- src/auth/ --passWithNoTests` exits 0
- `assert:coverage-threshold`: `jest --coverage --collectCoverageFrom="src/auth/**"` reports ≥80% lines
- `assert:no-skipped-tests`: output of `npm test -- src/auth/` contains zero "skipped" or "pending" entries

**Report to coordinator:** result of all three assertions (pass/fail per assertion, not overall pass/fail only)
```

Advantages: the coordinator receives structured per-assertion results, not just a binary pass/fail; failed assertions can be retried individually without rerunning the entire subagent; the AGENTS.md block serves as documentation for any human reviewing the pipeline. Use for complex subagent tasks where partial success is meaningful and retrying specific assertions is cheaper than rerunning the full task.

---

## The audit trail angle

In regulated environments — codebases subject to SOX, DORA, or internal change-control policies — every AI-assisted code change may need to be traceable to a human approval event. "The AI did it and I reviewed it" is not always sufficient. Some frameworks require evidence that the AI-produced change met defined criteria before it was merged.

Verifiable goals produce that evidence automatically. If your subagent's completion criterion is expressed as a set of named assertions, and each assertion run is logged — exit code, command, timestamp, output — you have a structured record that the change met its stated criteria before the coordinator proceeded.

Codex CLI observation hooks (v0.164.0 alpha) provide the logging mechanism. A `PostToolUse` observation hook that writes assertion results to a structured log file produces an audit artefact that survives the session.

The log entry for each assertion might look like:

```json
{
  "timestamp": "2026-10-21T03:47:22Z",
  "session_id": "codex-session-a4b7c2",
  "subagent_id": "auth-test-writer",
  "assertion": "assert:coverage-threshold",
  "command": "jest --coverage --collectCoverageFrom=\"src/auth/**\"",
  "exit_code": 0,
  "output_summary": "Lines: 84.2% (202/240)",
  "result": "pass"
}
```

This record tells a compliance auditor: before this change was accepted, the coverage criterion was checked, it passed, and here is the evidence. That is a qualitatively different audit posture from "an AI wrote some tests and a developer approved the PR."

---

## Failure modes: what happens when the criterion is ambiguous

Not every goal can be made fully verifiable, and some attempts to make goals verifiable introduce new failure modes.

**Over-specification.** A criterion that checks too many properties becomes brittle. If the criterion requires an exact line count, a specific function signature, and the presence of three named sections in a particular order, a subagent that produces excellent output in a slightly different structure fails the criterion and the coordinator halts — unnecessarily. Over-specified criteria produce false negatives that are annoying in development and expensive in production.

The fix: specify the minimum sufficient criterion. Check what must be true, not everything that is typically true. If a function's name is not load-bearing, do not assert the name.

**Criterion drift.** A criterion written for one version of the codebase may become incorrect as the codebase evolves. A coverage threshold that was reasonable at 60 files becomes too lenient at 200 files. A file existence check that assumed a specific directory structure fails when the structure is refactored.

The fix: treat completion criteria as first-class code. They live in AGENTS.md, they can be reviewed in PRs, and they should be updated when the codebase changes they depend on are merged.

**Verifying the wrong thing.** A criterion can pass while the underlying goal fails if the criterion is not well-matched to the goal. "Tests pass" passes if the tests are trivially true. "Coverage is 80%" passes if the coverage is over 80% for the wrong reasons.

The fix: compose criteria. A single criterion that checks one property is a proxy; two or three criteria that check different properties of the output are harder to satisfy without actually completing the goal. The three-assertion AGENTS.md pattern above is an example: each assertion checks a different property, and satisfying all three without doing the actual work is harder than satisfying any one of them.

---

## Connecting verifiable goals to the coordinator

A subagent that has checked its completion criterion needs to report the result to the coordinator in a form the coordinator can act on. Guo and Chua show this in the Command Centre view: each subagent reports a structured completion status — not just "done" but "done: all three assertions passed" or "done with failures: assert:coverage-threshold failed (74% < 80%)."

The coordinator can then decide: if all assertions passed, proceed to the next pipeline step. If an assertion failed, retry the subagent with the failure as context, escalate to a human review checkpoint, or halt the pipeline.

This is the connection between verifiable goals and auditable completion: the coordinator's decision log contains not just "subagent finished" but "subagent finished and met its stated criteria." When a pipeline run is reviewed after the fact — by a human auditor, by a post-mortem process, or by the developer who is debugging a production failure — the decision log explains why the coordinator proceeded at each step.

A coordinator that proceeds blindly — dispatches a subagent, receives "done," moves to the next step — is no more auditable than a one-shot prompt. A coordinator that proceeds conditionally — dispatches a subagent, receives structured assertion results, proceeds only when all assertions pass — is a pipeline you can reason about and a record you can defend.

---

## Getting started

The practical starting point is not a full multi-agent pipeline. It is one subagent task that you currently dispatch with a vague brief.

Pick a task your team runs regularly: "write tests for this module," "document this API," "refactor this function to handle the new interface." Write down what you would check to confirm the task was done correctly — the commands you run after you receive the output, the properties you look for in the files.

Convert those checks to a verifiable criterion in AGENTS.md. Run the subagent once with the criterion in place and manual oversight. If the subagent completes and the verification passes, you now have a task with a defined success condition. If the verification fails, you have discovered exactly what failed — not "the output looked off" but "the coverage check returned 74%."

Add an observation hook to log the assertion result. You now have an audit trail for this task.

Repeat for the next task.

The accumulation of verifiable subagent tasks is not a redesign of your workflow. It is a gradual replacement of "the AI did something and I reviewed it" with "the AI did something, confirmed it met these criteria, and here is the record."

---

[^1]: Charlie Guo and Gabriel Chua, "Cooking with Codex," AI Engineer World's Fair 2026, 2026-10-06. youtube.com/watch?v=FwiS44-xyYw. Guo is a Developer Experience engineer at OpenAI; Chua is a software engineer at OpenAI.

[^2]: Command Centre subagent view availability and feature set as shown in the AIEWF 2026 workshop demo. UI details may differ in current release versions. Refer to the Codex CLI changelog for current Command Centre capabilities.
