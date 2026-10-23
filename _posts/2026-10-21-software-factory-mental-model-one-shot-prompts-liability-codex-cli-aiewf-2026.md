---
title: "The Software Factory Mental Model: When One-Shot Prompts Become a Liability"
date: 2026-10-21T13:00:00+00:00
last_modified_at: 2026-10-23T10:06:16+01:00
parent: "Articles"
nav_order: 1197
tags:
  - codex-cli
  - agents
  - subagents
  - pipelines
  - agents-md
  - hooks
  - verifiable-goals
  - software-factory
  - aiewf-2026
  - workflows
copy_edited: true
---

# The Software Factory Mental Model: When One-Shot Prompts Become a Liability

![Sketchnote diagram for: The Software Factory Mental Model: When One-Shot Prompts Become a Liability](/sketchnotes/articles/2026-10-21-software-factory-mental-model-one-shot-prompts-liability-codex-cli-aiewf-2026.png)

**Source:** Charlie Guo & Gabriel Chua, "Cooking with Codex" workshop, AI Engineer World's Fair 2026 (youtube.com/watch?v=FwiS44-xyYw, 2026-10-06)
**Impact:** Any developer who has built a prompt they run repeatedly; teams scaling Codex CLI usage beyond one-off experiments

---

Charlie Guo opens the AIEWF 2026 workshop with a provocation that sounds like a slogan but is actually a structural claim: "The magic phrase is dead — long live the software factory."[^1]

The phrase lands as memorable friction. Most Codex users have spent meaningful effort crafting prompts that work: the right framing, the right examples, the right constraint language. The idea that this investment is a liability rather than an asset is uncomfortable. But Guo is not arguing against careful prompting. He is arguing against the *architecture* that makes the prompt the unit of reuse — and that argument holds up.

---

## The one-shot trap

A one-shot prompt is a prompt you run once, review the output, and either ship or discard. Used as an exploratory tool, it is entirely appropriate. The trap begins when a prompt that worked once becomes a process: something you or a team member run repeatedly, against different inputs, expecting reliable output.

The trap has three components.

**Fragility hidden by early success.** A prompt that produces excellent output on your initial test case may fail silently on edge cases you did not anticipate. Because the prompt appears to "work," there is no structural pressure to discover its boundaries. The first failure is often downstream — in production, in a code review, in a test suite — rather than at the point of generation.

**No defined completion criterion.** A prompt produces output. Whether that output is correct, complete, or safe depends entirely on the human reviewing it. This is not a problem when the review is careful and the stakes are low. It becomes a problem when the review is rushed, when the output is long, or when the reviewer does not have full context for what "correct" means. A prompt cannot tell you it has finished correctly. It can only stop generating tokens.

**Unscalable as a process asset.** A prompt is not a specification. You cannot hand it to a new team member and have them understand what the process is meant to produce, under what constraints, with what verification steps, and at what human-review checkpoints. A prompt is an instruction to a model. A process needs more than that.

Guo's diagnosis: if your Codex workflow depends on a particular prompt delivering a particular result, you have built a dependency on a fragile artefact that you cannot audit, cannot schedule reliably, and cannot hand to a subagent.

---

## The software factory alternative

The software factory mental model does not replace prompts. It contextualises them. In the factory model, a prompt is one step in a pipeline — a step with defined inputs, a defined task, a verifiable output, and a handoff to the next step or to a human reviewer.

Guo and Chua demonstrate this with a multi-step data processing pipeline at AIEWF. Each step does one thing. Each step has a machine-verifiable completion criterion. Each step knows, via AGENTS.md, what counts as done — not "did the model stop generating" but "does the output file exist, does `npm test` exit 0, does the linter pass?"

The practical difference is not cosmetic. A pipeline with verifiable steps at each stage can:

- **Fail fast at the right point.** If step two's output fails its criterion, the pipeline halts before step three begins. You discover the failure at step two, with the step-two context intact, rather than at the end of a twenty-step run.
- **Be handed to a subagent.** A coordinator agent can dispatch step two to a subagent with a clear brief and a clear criterion for success. The coordinator does not need to understand the implementation of step two; it needs to know whether step two succeeded.
- **Be scheduled.** A pipeline with defined steps and verifiable outputs can be run as a scheduled task — during off-hours, on a remote cloud session, without human oversight at every step. A one-shot prompt cannot be safely scheduled because it has no internal checkpoints and no criterion for success other than "the model returned something."
- **Be audited.** When something goes wrong in a production pipeline, you can inspect the output at each step, identify which step produced incorrect output, and replay that step with a corrected input or constraint. A one-shot prompt collapses the entire run into a single artefact.

---

## Identifying one-shot vs factory-ready uses

Not every Codex use case warrants a pipeline. The question is not "should I convert everything to a factory?" but "which of my current uses are carrying risks they should not be carrying?"

A simple diagnostic:

**One-shot uses that should stay one-shot:**
- Exploratory generation — drafting, brainstorming, first-pass synthesis where you expect to review everything and discard liberally
- Single-run tasks with a human reviewing 100% of the output before anything acts on it
- Tasks where the input is never repeated (a one-off data migration, a bespoke configuration file for a specific client)

**One-shot uses that are carrying unacknowledged pipeline risk:**
- Any prompt you or a team member runs more than once against different inputs expecting consistent results
- Any prompt whose output goes into production (code, config, documentation, data) without a defined review criterion — "I'll know it's wrong if I see it" is not a criterion
- Any prompt that forms part of a longer workflow, where its output is the input to the next step
- Any prompt you have added to a runbook, a script, or a CI job

The third category is the most dangerous. A prompt embedded in a script or a CI job has been operationalised without being factory-ised. It runs without a completion criterion, without internal checkpoints, and without a defined human-review interval. It is a one-shot prompt pretending to be a pipeline.

---

## The migration checklist

Guo and Chua present a five-element framework for converting a one-shot prompt into a factory step. Applied to Codex CLI, it looks like this:

**1. Define the input format precisely.**

A factory step takes a defined input. Not "some code" but "a TypeScript file in `src/` with no existing JSDoc comments." Not "the current task" but "a GitHub issue body with a reproduction case and an expected behaviour section."

Precision here serves two purposes: it constrains what the model is asked to do, reducing the probability of out-of-scope output, and it makes the step testable — you can generate synthetic inputs that probe edge cases before the step runs against real data.

In AGENTS.md terms: express this as a `context:` section at the top of the relevant task block. State what the step expects to receive, not just what it should produce.

**2. Decompose to single-responsibility steps.**

A step that does two things is harder to verify than a step that does one thing. If your prompt asks the model to "write the function and add tests for it," you now have two outputs to verify and two independent failure modes. Split them.

Guo's heuristic: if your prompt contains the word "and" more than once in describing the task, it is a candidate for decomposition.

**3. Specify a verifiable completion criterion.**

For each step, define what machine-verifiable evidence constitutes completion. Not "the code looks correct" but "`go test ./...` exits 0." Not "the documentation is present" but "every exported function in `src/api/` has a JSDoc block."

Codex CLI hooks make this checkable at the step boundary. A `PostToolUse` hook can run the verification command after the model's write action and return the result to the coordinator. If the verification fails, the step fails — and the pipeline does not proceed.

**4. Set the output verification scope.**

Not every step requires human review. Define explicitly which steps require human sign-off before the next step begins, and which can be verified entirely by machine and proceed automatically.

A step that writes to `tests/` may be safe to auto-verify and proceed. A step that writes to `src/core/auth/` should probably pause for human review before the downstream step runs. AGENTS.md `require-approval` blocks are the mechanism for expressing this — not as a default "ask about everything" posture, but as a deliberate, per-step decision.

**5. Record the supervision interval.**

For long-running or scheduled pipelines, define how often a human checks in on the pipeline state — not to review every output, but to confirm the pipeline is tracking as expected. Codex Cloud's observation hooks (introduced in v0.164.0 alpha) surface checkpoint events without requiring the human to be present at every step. A supervision interval of "check the observation feed every four steps" is better than "check when it finishes" — and much better than "check if anything goes wrong."

---

## The payoff

The framing shift from "prompt" to "pipeline step" is not about adding overhead to simple tasks. It is about changing what you can do with reliable Codex work once you have it.

A pipeline step can be:

- **Given to a subagent.** The Codex CLI Command Centre dispatches subagents with a brief and a completion criterion. If you have expressed your task as a factory step, you already have both. The subagent can run the step, verify the criterion, and report back to the coordinator — without the coordinator needing to understand the implementation.

- **Scheduled.** A step with defined inputs and a verifiable completion criterion can run as a scheduled cloud task. The cron fires, Codex picks up the input, runs the step, verifies the criterion, and either proceeds or pauses for human review — all without the laptop being open.

- **Audited after the fact.** Observation hooks write a structured event log. A pipeline of factory steps produces an audit trail: which step ran, what input it received, what output it produced, what verification command ran, and whether it passed. When something goes wrong in production at 03:00, you have a trace.

The one-shot prompt delivers its output and leaves no residue. The pipeline step delivers its output and leaves a record.

---

## Starting the transition

You do not need to convert your entire workflow on day one. Guo's recommendation at AIEWF is more modest: identify one recurring Codex use in your current workflow — one prompt you or your team runs more than twice a week — and convert it.

Write down the input format. Split it into single-responsibility steps if it currently does multiple things. Write a completion criterion you could express as a shell command. Decide whether each step needs human review before the next step begins, or whether machine verification is sufficient.

Add those decisions to AGENTS.md. Run the converted step once with manual oversight. If the verification command passes and the output is correct, you now have a factory step you can hand to a subagent, schedule, and audit.

Repeat for the next recurring use.

The magic phrase is not dead. But the pipeline that wraps it — with defined inputs, verifiable outputs, and deliberate supervision — is what makes it an asset rather than a dependency.

---

[^1]: Charlie Guo and Gabriel Chua, "Cooking with Codex," AI Engineer World's Fair 2026, 2026-10-06. youtube.com/watch?v=FwiS44-xyYw. Guo is a Developer Experience engineer at OpenAI; Chua is a software engineer at OpenAI.

[^2]: The workshop does not cite Torres et al. or Kang's GAIE model, but the three-element structure — preventive constraints, executable verification, human checkpoints — maps directly onto the supervision layers described independently in those papers. The convergence is noted here for the reader's reference; the article stands on the Guo/Chua workshop content alone.
