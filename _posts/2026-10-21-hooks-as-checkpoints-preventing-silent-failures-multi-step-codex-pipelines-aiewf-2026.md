---
title: "Hooks as Checkpoints: Preventing Silent Failures in Multi-Step Codex Pipelines"
date: 2026-10-21T14:00:00+00:00
last_modified_at: 2026-10-22T19:06:14+01:00
parent: "Articles"
nav_order: 1198
tags:
  - codex-cli
  - hooks
  - pipelines
  - agents
  - agents-md
  - v0.164.0
  - aiewf-2026
  - workflows
  - validation
copy_edited: true
---

# Hooks as Checkpoints: Preventing Silent Failures in Multi-Step Codex Pipelines

![Sketchnote diagram for: Hooks as Checkpoints: Preventing Silent Failures in Multi-Step Codex Pipelines](/sketchnotes/articles/2026-10-21-hooks-as-checkpoints-preventing-silent-failures-multi-step-codex-pipelines-aiewf-2026.png)

**Source:** Charlie Guo & Gabriel Chua, "Cooking with Codex" workshop, AI Engineer World's Fair 2026 (youtube.com/watch?v=FwiS44-xyYw, 2026-10-06)
**Impact:** Any developer running multi-step Codex pipelines; teams using Codex CLI for scheduled or overnight tasks where failures may go unnoticed

---

The most dangerous failure mode in a multi-step pipeline is not the one that crashes loudly. It is the one that completes quietly, produces output that looks plausible, and passes the results to the next step — where the error compounds.

Charlie Guo and Gabriel Chua address this directly in their AIEWF 2026 workshop. Their solution is not more careful prompting or more attentive human review. It is deterministic validators wired into the pipeline at each step boundary: hooks that fire at defined lifecycle points, run a verification command, and either pass the pipeline forward or halt it before the error propagates.[^1]

This article explains how that pattern works in Codex CLI, when to use it, and how to design hooks that are fast enough not to impede a long-running task but specific enough to catch real failures.

---

## What lifecycle hooks are in Codex

Codex CLI hooks are shell commands (or scripts) that Codex invokes at defined points in a task's execution lifecycle. As of v0.164.0 alpha, the supported hook types are:[^2]

- **`PreToolUse`** — fires before a tool call executes (file write, shell command, API call). Can inspect the intended action and, if configured as a blocking hook, prevent it from proceeding.
- **`PostToolUse`** — fires after a tool call completes. Receives the tool name, input parameters, and output. Useful for verifying the result of a specific action.
- **`OnSessionEnd`** — fires when the Codex session terminates, whether by task completion, error, or user interruption. Useful for cleanup and final state verification.
- **`Observation`** — introduced in v0.164.0 alpha; read-only hooks that receive event data but cannot gate the pipeline. Designed for logging, monitoring, and telemetry without introducing latency into the execution path.

The distinction between observation hooks and validation hooks is important. Observation hooks are safe to use freely — they add visibility without risk. Validation hooks (PreToolUse and PostToolUse) gate execution: a hook that exits non-zero signals failure, and Codex can be configured to halt the pipeline at that point. That gate is the checkpoint.

---

## The checkpoint pattern

Guo and Chua demonstrate the checkpoint pattern with a five-step data processing pipeline. Each step produces output that the next step depends on. At each step boundary, a `PostToolUse` hook runs a verification command — a shell command that returns 0 on success and non-zero on failure.

The structure looks like this:

```text
Step 1: Parse input → PostToolUse hook: verify output file exists + valid JSON
Step 2: Transform data → PostToolUse hook: run schema validation against output
Step 3: Write to staging → PostToolUse hook: run smoke test against staging endpoint
Step 4: Generate report → PostToolUse hook: verify report contains required sections
Step 5: Commit artefacts → PostToolUse hook: run git status, confirm expected files staged
```

Each hook is a single shell command. The pipeline cannot advance to step N+1 if step N's hook exits non-zero. The failure is surfaced at the exact point where the error occurred — not three steps later, when the corrupted data from step one has propagated through steps two, three, and four.

This is the core value of the checkpoint pattern: it converts a pipeline that can fail silently anywhere into a pipeline that fails loudly at the first boundary where the failure is detectable.

---

## A worked example

Consider a pipeline that processes a directory of Markdown files, converts them to HTML, runs a link checker against the output, and then deploys to a staging site. Without hooks, the pipeline runs start to finish. If the HTML conversion fails silently on one file — producing empty output rather than an error — the link checker runs against incomplete content and the deployment proceeds with a broken page.

With checkpoint hooks, each stage boundary has a verifier:

```bash
# PostToolUse hook after HTML conversion step
# Verifies that all .md input files have a corresponding non-empty .html output

#!/bin/bash
INPUT_DIR="${1:-src/content}"
OUTPUT_DIR="${2:-dist/html}"

md_count=$(find "$INPUT_DIR" -name "*.md" | wc -l)
html_count=$(find "$OUTPUT_DIR" -name "*.html" -size +0c | wc -l)

if [ "$html_count" -lt "$md_count" ]; then
  echo "HOOK FAIL: expected $md_count HTML files, found $html_count non-empty" >&2
  exit 1
fi

echo "HOOK PASS: $html_count/$md_count HTML files verified"
exit 0
```

This hook does not understand HTML. It does not validate the content. It asks one question — did every input file produce a non-empty output file? — and answers it with a shell command. If the answer is no, the pipeline halts before the link checker runs against incomplete output.

The hook is small. It runs in milliseconds. And it catches the silent failure before it propagates.

---

## Observation hooks vs validation hooks: choosing the right type

Not every hook should gate the pipeline. The choice between an observation hook and a validation hook depends on three factors:

**What you are checking.** If the check can definitively determine whether the step succeeded — a command exit code, a file existence check, a schema validation — use a validation hook. If the check is informational — logging step duration, recording the model's output for later review, sending a notification — use an observation hook.

**The cost of a false halt.** A validation hook that exits non-zero halts the pipeline. If your verification command is brittle — if it can fail for reasons unrelated to the step's correctness — a false halt is worse than no hook at all. Observation hooks carry no halt risk.

**The step's position in the pipeline.** Early-stage steps that produce intermediate artefacts used by later steps are the highest-value hook candidates. A failure at step two that is caught at step two saves the cost of running steps three through ten. A failure at the final step is less expensive to catch late, because you are not compounding wasted compute.

Guo's heuristic at AIEWF: put validation hooks on the steps whose failure would be hardest to diagnose from the final output. Put observation hooks everywhere else.

---

## Designing hooks that work at scale

A validation hook that takes thirty seconds to run adds thirty seconds to every step execution in a fifteen-hour pipeline. At ten steps, that is five minutes of overhead. At fifty steps across multiple subagents, it becomes meaningful. Hook design for long-running pipelines requires the same care as any other performance-sensitive code.

**Keep verification commands fast.** A hook that verifies a file exists and is non-empty runs in milliseconds. A hook that re-runs the full test suite runs in minutes. Reserve the expensive verification for the steps where only an expensive check can catch a real failure. Use cheap proxy checks — file size, line count, exit code — where they are sufficient.

**Make hooks specific to the step's output.** A generic hook that checks "did the model write any file?" is better than nothing but worse than a hook that checks "did the model write a file matching the expected schema?" Specificity reduces false negatives. A plausible-looking but incorrect output that passes a generic hook is still a silent failure.

**Distinguish between "step failed" and "pipeline should halt."** Some failures are recoverable: the model wrote an output file but it failed schema validation because of a missing optional field. A hook that exits non-zero on this condition halts the pipeline unnecessarily. Consider adding a retry signal — an exit code convention that Codex interprets as "retry this step with the validation failure as context" rather than "halt the pipeline." This is not a built-in Codex feature as of v0.164.0, but it can be implemented via hook exit code conventions documented in AGENTS.md.

**Log hook output.** A hook that exits non-zero without explanation leaves the operator with a halted pipeline and no diagnosis. Route hook output — both stdout and stderr — to a log file that survives the pipeline run. When a pipeline halts at 03:00, the first thing you want is the hook's output.

---

## Wiring hooks in AGENTS.md

Hooks are declared in `AGENTS.md` as part of the task configuration. A pipeline step that requires a `PostToolUse` verification hook looks like this:

```markdown
## Task: convert-markdown-to-html

**Input:** `src/content/*.md`
**Output:** `dist/html/*.html`
**Completion criterion:** All .md files have a corresponding non-empty .html output file.

### Hooks

- **PostToolUse** (blocking): `scripts/hooks/verify-html-output.sh src/content dist/html`
  - Exit 0: proceed to next step
  - Exit 1: halt pipeline, log failure to `logs/hook-failures.log`

### Notes

Do not proceed to the link-checking step if this hook exits non-zero. The link checker will produce misleading results against incomplete output.
```

The AGENTS.md declaration serves two purposes: it configures Codex to run the hook, and it documents the intent for any human reviewing the pipeline configuration. A pipeline operator looking at AGENTS.md should be able to understand, without reading the hook script, what the hook is checking and what the expected outcomes are.

---

## The difference the checkpoint pattern makes

A pipeline without checkpoints is a batch job: it runs, it produces output, and you discover failures by inspecting the final state — or by noticing downstream effects in production. The failure surface is the entire run.

A pipeline with checkpoints at each step boundary converts the failure surface to a single step. The pipeline halts at the first boundary where a failure is detectable. The operator's diagnosis task changes from "find where in this fifteen-hour run something went wrong" to "step four's hook exited non-zero — here is the hook output."

Guo and Chua's point at AIEWF is not that hooks eliminate failures. They do not. A hook can only detect the failure modes it was designed to detect. A step can produce output that passes all its hooks and still be subtly incorrect in ways that only a human reviewer will catch.

But a hook-gated pipeline surfaces detectable failures immediately, at the exact step where they occur, with structured output that explains what the hook checked and why it failed. That is a qualitatively different debugging experience from a silent failure that propagates through four subsequent steps before anything surfaces.

The checkpoint pattern does not make pipelines perfect. It makes failures findable.

---

[^1]: Charlie Guo and Gabriel Chua, "Cooking with Codex," AI Engineer World's Fair 2026, 2026-10-06. youtube.com/watch?v=FwiS44-xyYw. Guo is a Developer Experience engineer at OpenAI; Chua is a software engineer at OpenAI.

[^2]: Hook types and behaviour as of Codex CLI v0.164.0 alpha. The observation hook type was introduced in this release. Hook configuration syntax and exit code behaviour should be verified against the current Codex CLI changelog and AGENTS.md specification, as hook capabilities are an active development area.
