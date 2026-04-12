---
title: "SAFe Was Bad for Agility. For AI Coding Agents, It's Worse."
date: 2026-04-12T10:00:00+00:00
tags:
  - codex-cli
  - enterprise
  - safe
  - agile
  - process
  - adoption
  - opinion
---

![Sketchnote diagram for: SAFe Was Bad for Agility. For AI Coding Agents, It's Worse.](/sketchnotes/articles/2026-04-12-safe-enterprise-processes-ai-coding-agents.png)

Jeff Gothelf's recent article "[SAFe Was Bad for Agility. For AI, It's Catastrophic](https://jeffgothelf.com/blog/safe-was-bad-for-agility-for-ai-its-catastrophic/)" makes a sharp argument: the Scaled Agile Framework, which was already damaging for software agility, becomes actively destructive when applied to AI product development[^1]. His reasoning applies with equal force to enterprise adoption of AI coding agents like Codex CLI — and the implications for teams trying to get Codex CLI into production are worth examining carefully.

## The SAFe Problem, Briefly

SAFe (Scaled Agile Framework) is the dominant enterprise agile methodology. It organises work into Program Increments (PIs) — typically 8-12 week planning cycles with fixed scope, predetermined features, and hierarchical decision-making. In theory, SAFe provides predictability at scale. In practice, it replaces the responsiveness that agile was supposed to deliver with the rigidity that agile was supposed to eliminate.

Gothelf's argument is that AI development demands rapid experimentation, constant iteration based on model outputs, and the flexibility to pivot when approaches fail. SAFe's fixed planning cycles directly conflict with every one of these requirements[^1]. His concrete example is damning: an insurance company assembled a dedicated AI working group, ran three PI planning cycles with formally assigned AI use cases to Release Trains, produced a 21-slide deck explaining their AI strategy — and shipped zero AI-powered features in eight months. That, Gothelf writes, was "SAFe working exactly as designed."[^1]

He is not alone in this assessment. The AI4Agile Practitioners Report 2026 found that while 83% of Agile practitioners now use AI tools, most spend 10% or less of their time with them — integration uncertainty is the biggest adoption barrier at 54.3%, nearly 18 percentage points ahead of any other factor[^3]. The report reveals that organisations do not know *where AI fits* in their existing processes, and SAFe's rigid structure makes experimentation to find out logistically impossible.

## Why This Matters for Codex CLI in the Enterprise

The JetBrains survey data tells the story in numbers: Claude Code has 18% work adoption, while Codex CLI — despite having a larger total user base across the Codex platform — shows only 3% work adoption[^2]. The gap between "developers who have tried it" and "developers who use it at work" is not a product problem. It is a process problem.

Enterprise teams do not fail at AI coding adoption because the tools are insufficient. They fail because the processes surrounding the tools are designed for a world where humans write every line of code, every change goes through a predetermined review chain, and every sprint delivers pre-planned features.

The scale of this problem is staggering. A Writer survey found 79% of organisations face challenges in adopting AI — a double-digit increase from 2025 — with 54% of C-suite executives admitting that adopting AI is "tearing their company apart"[^4]. About 50% of agentic AI projects remain stuck in pilot stages, and Gartner predicts over 40% of agentic AI projects will be cancelled by end of 2027 due to escalating costs, unclear business value, or inadequate risk controls[^5]. Only 46% of developers trust AI tool output, and just 3% report "high trust"[^4].

Here is how SAFe-style processes specifically break AI coding agent workflows:

### 1. Fixed Sprint Scope Kills Exploration

Codex CLI's most powerful capability is exploratory: "Investigate this codebase and suggest an architecture for X." An agent might discover in the first 30 minutes that the planned approach is wrong and a completely different solution is needed. In a SAFe PI, that discovery creates a change request. The change request goes to a review board. The review board meets next week. The agent sits idle.

The entire point of AI coding agents is that they compress the feedback loop between "idea" and "working code" from days to minutes. SAFe stretches it back to weeks.

### 2. Predetermined Work Packages Prevent Agent Autonomy

SAFe decomposes work into stories, which are assigned to teams, which are scheduled into sprints. This works when humans need manageable chunks. It fails completely when an agent like Codex CLI can execute a 50-file refactor in 20 minutes that was planned as a two-sprint epic.

The agent does not care about your story point estimates. It does not respect sprint boundaries. It can do in one session what was planned for a month — but only if the process allows it to.

### 3. Approval Hierarchies Conflict with Agent Delegation

Codex CLI's approval policies (`auto-edit`, `full-auto`) are designed to let agents execute without constant human intervention. SAFe's hierarchical approval structures require human sign-off at every level. The result: teams configure Codex CLI to `suggest` mode (ask permission for everything), which strips the agent of its primary advantage and turns it into an expensive autocomplete tool.

The teams getting real value from Codex CLI are the ones running `full-auto` with guardrails (sandbox execution, git worktrees, automated tests). SAFe organisations rarely trust this approach because it bypasses the approval structures that justify the framework's existence.

### 4. Quarterly Planning Cannot Accommodate Weekly Tool Evolution

Codex CLI shipped version 0.119 on April 10 and version 0.120 on April 11. Features that did not exist on Monday are in production by Wednesday. SAFe's quarterly PI planning assumes a stable toolchain. When the toolchain evolves faster than the planning cycle, every PI is based on outdated assumptions about what is possible.

Teams locked into SAFe cannot adopt background agent streaming (v0.120.0), subagent parallelism, or new MCP capabilities without waiting for the next PI boundary — by which time three more versions have shipped.

### 5. SAFe's Quality Model Assumes Human Authors

SAFe's Built-in Quality relies on developer discipline, craft values, and professional pride. AI agents have none of these. A detailed analysis from Agility at Scale found that AI agents generate code at 98% higher volume while producing 1.7× more defects than human developers[^6]. CodeRabbit data shows agents produce 2.74× more cross-site scripting vulnerabilities and 8× more performance problems. Review capacity cannot keep up: teams require 91% more review time despite the higher defect rates[^6].

The solution is not to slow agents down — it is to replace craft-based quality with mechanically enforced governance. Harness engineering — instruction contracts, MCP servers, skills, hooks, and back-pressure mechanisms — encodes quality standards as machine-readable policy gates rather than human guidelines[^6]. Codex CLI already supports this architecture through `codex.toml` profiles, sandbox constraints, and hook-based validation. SAFe's quality model does not account for any of it.

### 6. The AI Maturity Gap Is a SAFe Structural Problem

An AI maturity framework for SAFe enterprises identifies four assessment dimensions: team structure fitness, delivery pipeline readiness, quality practice maturity, and AI infrastructure readiness. The critical finding: "Aggregate maturity equals the minimum of the four dimension scores" — a single weak dimension constrains overall readiness regardless of strength elsewhere[^7].

Most SAFe organisations score well on team structure but poorly on AI infrastructure and quality automation. The hardest transition — Level 2 (AI-Augmented, individual productivity gains) to Level 3 (AI-Integrated, organisational adaptation) — requires coordinated structural changes across all four dimensions simultaneously[^7]. SAFe's Release Train structure makes this coordination nearly impossible because each train optimises locally rather than systemically.

The report identifies a paradox that directly mirrors the Codex CLI adoption gap: individual developers show +98% more PRs merged, but organisational delivery metrics remain flat because downstream review capacity cannot absorb the volume[^7].

## What Enterprise Processes Need Instead

Gothelf argues for replacing SAFe with experimentation-driven approaches for AI work. In a companion piece, "[Agile in the Age of AI](https://jeffgothelf.com/blog/agile-in-the-age-of-ai/comment-page-1/)," he makes the case that agile itself survives the AI era — but only if it returns to its original principles of rapid iteration and continuous learning, not the ceremonial overhead that SAFe has layered on top[^8]. For enterprise Codex CLI adoption specifically, this translates to:

### Outcome-Based Work, Not Task-Based Work

Instead of "Implement story JIRA-1234: Add validation to the user form," the work unit becomes "Reduce form submission errors by 40%." The agent decides how to achieve the outcome. The human reviews the result, not the approach.

This maps directly to Codex CLI's strengths. Give the agent a goal and a test suite. Let it explore solutions. Review the pull request, not the individual keystrokes.

### Continuous Review, Not Batch Review

Replace sprint reviews with continuous PR-based review. Codex CLI generates pull requests with explanations. Reviewers assess the PR on its merits — does it work, is it safe, does it meet standards — regardless of whether it was planned for this sprint.

This is how every high-performing Codex CLI team already works. The process needs to catch up with the tool.

### Trust Boundaries, Not Approval Chains

Instead of hierarchical approvals, define trust boundaries:

```toml
# codex.toml - enterprise trust configuration
[profiles.safe-refactor]
model = "gpt-5.4"
approval_policy = "auto-edit"
sandbox = true
writable_paths = ["src/", "tests/"]

[profiles.infrastructure]
model = "gpt-5.4"
approval_policy = "suggest"
writable_paths = ["infra/"]
```

The agent operates freely within defined boundaries. Code in `src/` and `tests/` gets auto-edited with sandbox protection. Infrastructure changes require human approval. The trust model is encoded in configuration, not in meeting invitations.

### Weekly Capability Reviews, Not Quarterly Planning

Replace PI planning with weekly 30-minute capability reviews: what has the toolchain shipped this week? What new patterns are available? What experiments should we run? This keeps the team's mental model aligned with the tool's actual capabilities rather than the capabilities it had three months ago.

## The Adoption Gap Is a Process Gap

The 3% work adoption figure for Codex CLI is not a ceiling imposed by the technology. It is a ceiling imposed by enterprise processes that were designed for a different era. Codex CLI can already do the work. The question is whether enterprises will redesign their processes to let it.

Gothelf's conclusion about AI generally applies to AI coding agents specifically: frameworks built for predictability become obstacles when the work demands adaptability. SAFe gave enterprises the illusion of agility. For AI coding agents, it gives them the illusion of adoption — teams that have the tools but cannot use them because the process will not allow it.

The enterprises that succeed with Codex CLI will be the ones that treat process change as a prerequisite for tool adoption, not a follow-up task.

---

## Citations

[^1]: Jeff Gothelf, [SAFe Was Bad for Agility. For AI, It's Catastrophic](https://jeffgothelf.com/blog/safe-was-bad-for-agility-for-ai-its-catastrophic/), April 2026. Argues SAFe's fixed planning cycles, hierarchical decision-making, and predetermined scope are fundamentally incompatible with AI's demand for rapid experimentation. Includes insurance company case study: 8 months, 3 PI cycles, 21-slide deck, zero shipped features.

[^2]: JetBrains Research, "Which AI Coding Tools Do Developers Actually Use at Work?" April 2026. Survey of 10,000+ developers. Claude Code 18% work adoption, Codex CLI 3%, Google Antigravity 6%.

[^3]: DZone, [The AI4Agile Practitioners Report 2026](https://dzone.com/articles/ai4agile-practitioners-report). 83% of Agile practitioners use AI, but most spend ≤10% of their time with it. Integration uncertainty is the top barrier at 54.3%.

[^4]: Writer, [Enterprise AI Adoption in 2026: Why 79% Face Challenges Despite High Investment](https://writer.com/blog/enterprise-ai-adoption-2026/). 79% of organisations face AI adoption challenges; 54% of C-suite say AI is "tearing their company apart." 46% of developers distrust AI output; only 3% report "high trust."

[^5]: Gartner, [Predicts 2026: AI Agents Will Reshape Infrastructure & Operations](https://www.itential.com/resource/analyst-report/gartner-predicts-2026-ai-agents-will-reshape-infrastructure-operations/). Over 40% of agentic AI projects will be cancelled by end of 2027. Only 11% of organisations had deployed agentic AI by mid-2025 despite 99% planning to eventually.

[^6]: Agility at Scale, [SAFe Built-in Quality When AI Agents Write the Code](https://agility-at-scale.com/safe/safe-built-in-quality-when-ai-agents-write-the-code/). AI agents produce 98% more code volume with 1.7× more defects. 2.74× more XSS vulnerabilities. 8× more performance problems. 91% more review time required. Proposes harness engineering with 6-layer governance.

[^7]: Agility at Scale, [AI Maturity for SAFe Enterprises: The Missing Integration Framework](https://agility-at-scale.com/safe/ai-maturity-for-safe-enterprises-the-missing-integration-framework/). Four-dimension assessment model. Five maturity levels (AI-Aware → AI-Native). Aggregate maturity = minimum of four dimension scores. L2→L3 transition is the hardest, requiring coordinated structural change.

[^8]: Jeff Gothelf, [Agile in the Age of AI](https://jeffgothelf.com/blog/agile-in-the-age-of-ai/), December 2025. Argues agile survives the AI era but must return to original principles of rapid iteration, not ceremonial overhead.
