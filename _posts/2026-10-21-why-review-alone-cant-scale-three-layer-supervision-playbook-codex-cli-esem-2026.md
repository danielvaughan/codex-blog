---
title: "Why Review Alone Can't Scale: The Three-Layer Supervision Playbook"
date: 2026-10-21T10:00:00+00:00
last_modified_at: 2026-10-27T15:05:40Z
parent: "Articles"
nav_order: 1195
tags:
  - codex-cli
  - supervision
  - code-review
  - agents
  - hooks
  - agents-md
  - guardian
  - ci-cd
  - governance
  - esem-2026
copy_edited: true
---

# Why Review Alone Can't Scale: The Three-Layer Supervision Playbook

![Sketchnote diagram for: Why Review Alone Can't Scale: The Three-Layer Supervision Playbook](/sketchnotes/articles/2026-10-21-why-review-alone-cant-scale-three-layer-supervision-playbook-codex-cli-esem-2026.png)

**Research basis:** Stolze and Strässle "When Review Alone No Longer Scales" (arXiv:2608.26316, ESEM 2026); Kang "GAIE Oversight Classification Model" (arXiv:2606.22484)
**Impact:** All teams using Codex CLI or AI-assisted development at scale; engineering leads designing review processes

---

There is a throughput problem at the centre of AI-assisted software engineering, and most teams are trying to solve it the wrong way. When AI tools write more code faster, the instinctive response is to add more reviewers, or ask reviewers to work faster, or lower the review bar for "routine" changes. None of these responses work at scale, and Stolze and Strässle's ESEM 2026 study explains precisely why.[^1]

The finding is not that human review is insufficient — it is that human review is being asked to absorb a structural load it was never designed to carry. The solution is not to strengthen the single layer of human review but to reduce what reaches it.

---

## The volume problem is structural, not a skills gap

Stolze and Strässle conducted practitioner interviews and a follow-up survey across engineering organisations that had adopted AI coding assistants. The central finding is consistent across organisation size: when AI-generated output volume increases by an order of magnitude, review throughput does not increase proportionally. It cannot. Human reviewers have bounded attention, bounded context windows, and bounded time. The bottleneck is architectural, not individual.

Three specific failure patterns emerge from the study data:

**Attention dilution.** As the volume of AI-generated PRs increases, reviewers spend less time per change. Reviewers reported consciously or unconsciously raising the threshold for what they flag — "I stopped questioning the boilerplate because there was too much of it." Subtle architectural drift and naming convention violations pass through because the reviewer's attention is consumed by the volume of clearly correct code surrounding them.

**Context fragmentation.** AI coding tools generate changes at a pace that prevents reviewers from maintaining a coherent mental model of the evolving codebase. Reviewers are evaluating individual diffs rather than trajectories. A sequence of individually-reasonable changes that collectively shift an architectural invariant is invisible to a reviewer working at this pace.

**Reviewer fatigue at the merge gate.** When all supervision is concentrated at the pull-request stage, reviewers become the sole quality gate for a high-velocity pipeline. Fatigue sets in. The review bar degrades precisely when it is under maximum load — a perverse outcome that the study documents empirically.

The conclusion is not that humans should review less carefully. It is that concentrating all supervision at the merge gate is a design flaw. The volume of AI-generated code that needs to reach a human reviewer must be reduced before it arrives.

---

## The three-layer model

Stolze and Strässle propose a three-layer supervision structure. Kang's GAIE Oversight Classification Model (arXiv:2606.22484) independently reaches the same three-layer structure from a different research programme — a convergence that signals a stable finding rather than a single team's preference.[^2]

The layers are:

**Layer 1 — Preventive guardrails.** Machine-readable rules that agents follow during generation. These are constraints expressed in AGENTS.md, named profiles, model directives, and tool permission configurations. They prevent problematic output from being generated in the first place. A rule that says "never modify files in `src/core/` without explicit approval" does not review a PR; it prevents the PR from existing. A directive that says "always run `npm test` before completing a task" does not catch a failing test at review time; it prevents the task from completing with a failing test.

Preventive guardrails reduce the volume of problematic output before any code is written.

**Layer 2 — Executable guardrails.** Automated checks that run against generated output before it reaches a human reviewer. This layer repurposes existing infrastructure — testing pipelines, linters, type checkers, security scanners, CI/CD gates — as a scalable supervision layer. The key reframe is that these tools are not optional quality improvements; they are the primary machine-layer supervisor for AI-generated code.

Stolze and Strässle find that teams treating Layer 2 as mandatory and blocking — not advisory — reduce the defect rate at the human review stage significantly. The reviewers who participate in their study consistently report that reviewing AI-generated code that has passed a comprehensive automated gate is qualitatively different from reviewing code that has not: "I can focus on what the tool can't check. Whether this approach makes architectural sense. Whether we'll regret the abstraction in six months."

**Layer 3 — Human oversight focused on judgment, not correctness.** With Layers 1 and 2 absorbing routine correctness verification, human review can be reoriented toward what only humans can assess: architectural coherence, long-term maintainability, alignment with product direction, and the second-order effects of repeated patterns. This is not a downgrade of human review — it is a precision upgrade. The reviewer's attention is deployed where it has comparative advantage.

---

## Implementing the three layers in Codex CLI

### Layer 1: Preventive guardrails in AGENTS.md

AGENTS.md is the primary preventive guardrail surface in Codex CLI. Key patterns for high-volume environments:

**Explicit scope boundaries:**
```markdown
## Modification permissions
- `src/core/`: requires explicit instruction per file; never modify on initiative
- `src/api/`: requires `approval_policy: full` for any structural change
- `tests/`: always update tests when modifying logic; never reduce coverage
- `package.json`: never add or remove dependencies without explicit instruction
```

**Completion criteria that prevent incomplete work from surfacing:**
```markdown
## Task completion standards
A task is not complete until:
- [ ] All modified files have a corresponding test
- [ ] `npm test` exits 0
- [ ] `npm run lint` exits 0
- [ ] A TASK-SUMMARY.md is written to the repo root describing what changed and why
```

**Named profiles for review-sensitive contexts:**
```yaml
# codex/profiles/pr-ready.yaml
name: pr-ready
description: "Use when preparing work for PR submission"
approval_policy: full
model_reasoning_effort: high
directives:
  - "Do not proceed past any step where confidence is below 90%"
  - "Add inline comments explaining any non-obvious decision"
  - "Flag any change to public API surfaces for explicit human review before committing"
```

### Layer 2: Executable guardrails as mandatory gates

The reframe here is from "CI that runs after the PR is opened" to "blocking gate before the task completes." PostToolUse hooks in Codex CLI make this concrete:

```json
{
  "hooks": {
    "postToolUse": [
      {
        "trigger": "write_file",
        "command": "npm run lint -- --fix-dry-run ${CODEX_TOOL_PATH}",
        "on_failure": "block",
        "message": "Lint check failed. Fix violations before proceeding."
      },
      {
        "trigger": "bash",
        "pattern": "git commit",
        "command": "npm test",
        "on_failure": "block",
        "message": "Tests must pass before commit."
      }
    ]
  }
}
```

For security-sensitive codebases, a PreToolUse hook that runs a static analysis scan before any file write adds a second machine-layer check at the point of generation rather than after:

```bash
#!/bin/bash
# Pre-write security scan hook
if echo "$CODEX_TOOL_PATH" | grep -qE "\.(js|ts|py)$"; then
  semgrep --config=p/default "$CODEX_TOOL_PATH" 2>/dev/null
  if [ $? -ne 0 ]; then
    echo "Security scan flagged this file. Review before writing." >&2
    exit 1
  fi
fi
```

The Guardian subagent pattern takes Layer 2 further: a dedicated Codex subagent whose sole role is to review the primary agent's output against a checklist before it is committed. The Guardian has no write permissions; it evaluates and approves or blocks. This adds a second AI-layer review without human intervention and is well-suited for organisations that want audit evidence of multi-stage review.

### Layer 3: Structuring human review for judgment

With Layers 1 and 2 operational, the PR description becomes the human reviewer's primary tool for focusing attention on what matters. A TASK-SUMMARY.md written by the agent as part of task completion should answer:

- What changed and why
- What was explicitly deferred and why
- What architectural assumptions the change relies on
- Which automated checks passed and which required manual resolution

This shifts the reviewer from "verify correctness" (Layer 2's job) to "assess judgment" — whether the agent's architectural reasoning holds, whether the deferred work creates risk, whether the approach is the right one.

Stolze and Strässle observe that this reorientation has a secondary benefit: it makes review more satisfying. Reviewers reported feeling more engaged when their role was to evaluate reasoning rather than check for bugs that a linter should have caught.

---

## Where the model breaks down

The three-layer model depends on the quality of Layer 1 and Layer 2 implementations. Three common failure modes:

**Underspecified AGENTS.md.** Preventive guardrails that are too vague ("be careful with core files") provide no machine-enforceable constraint. Preventive guardrails must be specific enough to be evaluated programmatically by the agent — either the condition is met or it is not.

**Advisory CI rather than blocking CI.** If Layer 2 gates are warnings rather than blockers, they are absorbed into the attention-dilution problem they are meant to solve. A failed lint warning in a sea of green CI checks is invisible at review time. Every Layer 2 gate that should block must block.

**Layer 3 reviewers reverting to Layer 2 habits.** If reviewers have been trained by years of catch-everything review to check for correctness themselves, shifting to judgment-focused review requires deliberate process change. The payoff is real — Stolze and Strässle document a measurable increase in architectural defect detection when reviewers operate at Layer 3 — but the transition is not automatic.

---

## The independent convergence with GAIE

The fact that Kang's GAIE Oversight Classification Model arrives at the same three-layer structure from a purely theoretical direction — classifying oversight mechanisms by their position in the generation-evaluation pipeline — strengthens the practical case for the Stolze–Strässle model.[^2]

GAIE's terminology differs (upstream constraints, inline validation, downstream verification) but the structure is identical. Two independent research programmes, one empirical and one theoretical, pointing to the same architecture is as close to a stable finding as software engineering research produces. Teams designing supervision infrastructure for AI-assisted development have empirical and theoretical grounds to treat the three-layer model as a reasonable default rather than a speculative one.

---

## The practical transition

For teams currently running single-layer review, the transition to three layers does not require a wholesale rewrite of processes. The incremental path:

1. **Audit what Layer 2 infrastructure you already have.** Most codebases have linters, type checkers, and test suites. The gap is usually that these run after PR creation rather than as blocking gates before the agent completes a task.

2. **Add PostToolUse blocking hooks for the most common defect categories.** Start with lint and test. Add security scanning for security-sensitive paths. Each hook reduces the volume reaching human review.

3. **Write an AGENTS.md section for modification permissions.** Identify the five files or directories in your codebase where a subtle mistake would have the highest impact. Add explicit guardrails for those paths first.

4. **Rewrite your PR template.** Add a section requiring the agent (or developer) to articulate the architectural reasoning behind the change, not just the list of what changed. This trains both the agent and the reviewers to operate at Layer 3.

5. **Measure the shift.** Track the defect categories that reviews catch before and after. If Layer 2 is absorbing correctness defects, Layer 3 should be surfacing more architectural ones. If the mix does not shift, the layers are not working as intended.

The goal is not to remove humans from review — it is to ensure that human attention is spent where it has the highest value. Stolze and Strässle put it plainly: the question is not whether to trust AI-generated code, but where in the pipeline human judgment is most irreplaceable.[^1]

---

## References

[^1]: Stolze, M. and Strässle, M. "When Review Alone No Longer Scales: Layered Supervision in AI-Assisted Software Engineering." arXiv:2608.26316, August 2026. Accepted at ESEM 2026. <https://arxiv.org/abs/2608.26316>
[^2]: Kang, J. "GAIE Oversight Classification Model: A Framework for Positioning Supervision Mechanisms in AI-Assisted Engineering Pipelines." arXiv:2606.22484, June 2026. <https://arxiv.org/abs/2606.22484>
