---
title: "The Codex Skill Flywheel: How Defined Skills Compound Agent Capability Over Time"
date: 2026-10-22T13:00:00+00:00
last_modified_at: 2026-10-28T03:09:23Z
parent: "Articles"
nav_order: 1198
tags:
  - codex-cli
  - skills
  - agents-md
  - context-engineering
  - workflow
  - knowledge-management
  - compounding
copy_edited: true
---

# The Codex Skill Flywheel: How Defined Skills Compound Agent Capability Over Time

![Sketchnote diagram for: The Codex Skill Flywheel: How Defined Skills Compound Agent Capability Over Time](/sketchnotes/articles/2026-10-22-codex-skill-flywheel-compounding-capability-agent-knowledge-codex-cli.png)

**Pattern:** Skills as compounding knowledge artefacts
**Mechanism:** Each skill reduces prompt overhead, enabling more skills
**Risk:** Skill sprawl — addressed by the flywheel model itself
**Category:** Context engineering, workflow architecture

---

Most teams discover skills in Codex CLI the way they discover git hooks: by noticing that they keep typing the same instructions, and eventually deciding to stop.

That is a reasonable origin story, but it undersells what is actually happening when you define a skill. A skill is not just a saved prompt. It is a unit of institutional memory that changes the cost structure of future work — including the work of building more skills. This compounding effect is what I mean by the skill flywheel, and once you see it, it changes how you think about the economics of context engineering.

---

## What Skills Actually Are

In Codex CLI, a skill is a Markdown file placed in a `skills/` directory (typically `.codex/skills/` at the project root, or the global skills directory for cross-project capabilities). When invoked — either explicitly by the user or automatically by a hook — the skill's content is injected into the agent's context at the point of invocation.

The structure is deliberately minimal:

```markdown
# Deploy to Staging

1. Run `npm run build` and confirm exit 0
2. Run `npm run test:integration` — abort if any failures
3. Push to the `staging` branch; do not force-push
4. Post the deploy SHA to #deployments in Slack via MCP
5. Wait for the smoke test hook to confirm green before proceeding
```

That is a complete skill. It encodes four things that would otherwise have to live in the user's prompt, in the agent's context window, or — worst — in someone's memory:

1. The correct sequence of operations
2. The conditions for aborting
3. The constraints on the operation (no force-push)
4. The side effects expected by the broader system (Slack notification)

Each of those four things has a cost if it is absent. Wrong sequence means broken deploys. Missing abort conditions means a bad build reaching staging. Absent constraints mean force-push incidents. Missing side effects mean Slack falling silent during a deploy that the rest of the team is watching.

---

## The Flywheel Mechanism

The flywheel starts slowly and accelerates. Here is the pattern across the first few months of a typical team's Codex CLI adoption:

**Month 1 — Discovery.** Skills are defined reactively, in response to repeated prompts. The team creates three or four skills: `run-tests`, `deploy-staging`, `open-pr`. Each one is slightly messy because the team is still learning what Codex needs to know versus what it already knows. The skills work, but they require occasional correction.

**Month 2 — Stabilisation.** The team notices that correcting a skill is cheaper than correcting an agent mid-task. A skill fix is a file edit; a mid-task correction is a multi-turn conversation with uncertain outcome. They start writing skills more deliberately, with explicit abort conditions and constraint lists. The skills become reliable enough to invoke without supervision.

**Month 3 — Composition.** Skills start referencing each other. The `release` skill calls the `run-tests` skill, then the `deploy-staging` skill, then the `open-pr` skill. What was three separate invocations becomes one. The agent navigates a complex, multi-step workflow with a single instruction because the workflow knowledge is encoded in the skill graph, not the prompt.

**Month 4 — Self-improvement.** The team starts using skills to define skills. A `write-skill` skill encodes the team's conventions for skill authorship — the abort-condition pattern, the constraint list format, the MCP notification hooks that always need to be included. New skills written with the help of this meta-skill are better from the start than the first-month skills written without it.

This is the flywheel: each skill reduces the cognitive overhead of the next skill, which makes skills cheaper to write, which increases the rate of skill creation, which reduces overhead further.

---

## Why the Flywheel Works

The underlying mechanism is context leverage. Codex CLI's context window is finite. Every token spent on instructions is a token not spent on code, diffs, test output, or error messages. Skills shift the instruction cost from runtime (paid on every invocation) to authorship time (paid once, amortised across all future invocations).

The maths of this amortisation improve non-linearly as the skill graph grows. A skill that takes 500 tokens to encode but is invoked 50 times saves 25,000 tokens of runtime context over its lifetime — context that can instead hold 25,000 tokens of more relevant information. Multiplied across a skill library of 30 or 40 skills, each invoked regularly, the cumulative context savings are substantial.

But context savings are only part of the story. The more important compounding effect is correctness. An agent operating from a well-defined skill has a smaller decision surface. The skill has pre-made certain decisions — sequence, constraints, abort conditions — that the agent would otherwise have to derive from the prompt. Fewer live decisions means fewer live errors. The agent's behaviour becomes more predictable, which makes it easier to write tests, hooks, and further skills around it.

---

## The Skill Authorship Gradient

Not all skills are equal. There is a gradient of skill quality that becomes visible once you have been running a flywheel for a few months:

**Level 1 — Sequence skills.** A list of steps. Useful, but fragile: they encode what to do but not when to stop, what to verify, or what to do when something goes wrong. Most first-month skills live here.

**Level 2 — Constrained skills.** Steps plus explicit constraints (no force-push, no changes to `main`, always run tests first). These survive contact with unusual situations better than Level 1 skills because the constraints rule out common failure modes before the agent has to reason about them.

**Level 3 — Verified skills.** Steps, constraints, and verification gates. Each significant step includes a check that confirms the step succeeded before proceeding. These skills are safe to invoke unattended. The agent will abort with a clear error rather than silently proceeding through a failure.

**Level 4 — Composable skills.** Skills that call other skills. The authorship here focuses on orchestration: what to do when a sub-skill fails, how to handle partial completion, how to surface status to the human. These skills encode workflow knowledge that is genuinely difficult to express in a one-shot prompt.

The flywheel accelerates most visibly at Level 3 and Level 4. Teams that only ever write Level 1 skills find that skills help but do not compound — each skill is standalone, and the skill library does not get smarter over time. Teams that invest in Level 3 and Level 4 find that the library starts to feel like a second agent: it makes the primary agent more capable without adding to the human's cognitive load.

---

## Skill Sprawl: The Flywheel's Failure Mode

The flywheel has one significant failure mode: skill sprawl.

Skills are easy to create and easy to forget. A team that writes skills reactively, without a curation practice, ends up with a library where:

- Multiple skills cover overlapping territory, creating inconsistency
- Old skills reference tool versions, API endpoints, or workflow steps that have changed
- The skill graph has orphaned nodes — skills that were once called by others but whose callers have been updated
- No one knows which skills are authoritative and which are deprecated

Skill sprawl reverses the flywheel. Instead of reducing cognitive overhead, the skill library adds it: the human must now choose between competing skills, verify that the skill they are invoking is current, and manage the risk that the agent will use an outdated skill from context.

The remedy is a curation skill. A `review-skills` skill that, when invoked, audits the skill library against the current workflow, flags overlapping or outdated entries, and proposes consolidations. The curation skill is itself a Level 4 skill — one that uses the agent to maintain the agent's own knowledge base. This is the clearest expression of the flywheel: the system that skills created is maintained by skills.

---

## The AGENTS.md Connection

Skills do not exist in isolation. The skill flywheel operates within the broader context of `AGENTS.md`, which governs how Codex CLI behaves across the entire repository. The relationship matters for two reasons.

First, `AGENTS.md` can declare which skills are available and when they should be invoked automatically (via hooks), versus manually. This makes the skill graph explicit and auditable. A reviewer reading `AGENTS.md` can understand the full capability surface of the agent without having to read every skill file.

Second, `AGENTS.md` can encode the meta-conventions of skill authorship — the Level 2 and Level 3 patterns — so that new skills written by the agent or by new team members inherit the right quality baseline from the start. The skill flywheel and the `AGENTS.md` governance layer are complementary: one provides capability, the other provides constraint.

---

## Starting the Flywheel

For teams not yet using skills deliberately, the entry point is a single question: what instruction do you find yourself giving Codex most often?

That instruction is your first skill. Write it as a Level 2 skill — steps plus constraints — and invoke it for a week. At the end of the week, edit it based on what went wrong or what felt missing. After three or four edit cycles, it will be a reliable Level 3 skill.

From there, look for the second most common instruction. Write it the same way. Then look for the first place the two skills could be composed.

The flywheel does not require a library of thirty skills to start spinning. It requires two skills, a willingness to edit them, and a month of consistent use. The compounding takes care of itself.

---

## Summary

| Level | Skill type | Characteristic | Failure mode |
|---|---|---|---|
| 1 | Sequence | Steps only | Fragile on edge cases |
| 2 | Constrained | Steps + constraints | Verbose but reliable |
| 3 | Verified | Steps + constraints + gates | Abort-on-failure, safe unattended |
| 4 | Composable | Calls other skills | Requires curation to stay healthy |

The skill flywheel is not a feature of Codex CLI. It is an emergent property of using skills consistently over time, with enough discipline to edit them and enough structure to compose them. The teams that get the most from Codex are not necessarily the ones with the most skills — they are the ones whose skills get better faster.
