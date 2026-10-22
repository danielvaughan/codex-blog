---
title: "GPT-5.5 Is Gone: What It Actually Meant for Codex CLI Workflows"
date: 2026-10-14T08:00:00+00:00
last_modified_at: 2026-10-22T08:20:41+01:00
parent: "Articles"
nav_order: 1181
tags:
  - codex-cli
  - openai
  - model-selection
  - gpt-5
  - gpt-6
  - migration
  - ci-cd
  - developer-experience
  - model-retirement
---

# GPT-5.5 Is Gone: What It Actually Meant for Codex CLI Workflows

![Sketchnote diagram for: GPT-5.5 Is Gone: What It Actually Meant for Codex CLI Workflows](/sketchnotes/articles/2026-10-14-gpt-55-retirement-what-it-meant-codex-cli-workflows.png)

This morning, GPT-5.5 was removed from ChatGPT, ChatGPT Work, and Codex on every plan.[^1] OpenAI marked the occasion with an emoji.[^2] The community marked it with something closer to a wake.

I want to write about what GPT-5.5 actually meant for Codex CLI workflows — not the official narrative, which is frictionless, but the one playing out in CI pipelines, code-review gates, and the GitHub Discussions threads where developers describe spending their Tuesday fixing scheduled jobs that no longer run.

## What Retired Today

GPT-5.5 was the model that most Codex CLI teams settled on over the twelve months before GPT-6 arrived. Not because it was the best available — it wasn't — but because it was predictable. It had a known cost profile, a known latency envelope, and a known failure mode. For teams building automated workflows, predictability is often more valuable than peak capability.

The practical fingerprint of GPT-5.5 across Codex installations was surprisingly large. It appeared in:

- `config.yaml` `model:` fields set once and never revisited
- AGENTS.md headers specifying `model: gpt-5.5` for cost-sensitive tasks
- CI job definitions in `.github/workflows/` passing `--model gpt-5.5`
- Code-review gates and scheduled security scans configured when GPT-5.5 was the best available option at a given price
- Plugin manifests declaring GPT-5.5 as the preferred execution model for low-complexity tasks

None of these configurations fails gracefully. When a CI job runs tonight referencing a retired model, it will fail. The error messages are not always informative.[^3] The AGENTS.md files and config files sit in version-controlled repositories, meaning the fix requires a commit, a PR, a review, and a deploy — the full change pipeline — for what is, structurally, a forced migration that developers did not initiate.

## The Pricing Asymmetry

OpenAI's official migration recommendation is GPT-6.1 Sol or GPT-6 Astra.[^1] The guidance is technically accurate. It is also incomplete.

GPT-6.1 Sol costs approximately 2.25× more per token than GPT-5.5 did.[^4] For teams running Codex in CI at high frequency — automated test generation, scheduled security scans, continuous code-review pipelines — this is not a small difference. It is the difference between a comfortable budget allocation and one that needs a conversation with an engineering manager.

GPT-6 Astra costs more still, and provides capabilities most CI automation tasks do not require. Sending a nightly dependency-upgrade agent through Astra is the model-selection equivalent of taking a Ferrari to the supermarket.

What OpenAI has not promoted heavily is Terra: described in their documentation as "competitive with GPT-5.5 at lower cost".[^4] Community members who found this comparison discovered it largely through their own research, not through the official retirement guidance. A formal request for a GPT-5.5 legacy option — filed with OpenAI in September with documented support from teams running large Codex deployments — was rejected.[^5]

This is not unusual behaviour in the model landscape. Google deprecated PaLM 2 on a similar timeline. Anthropic has cycled Claude generations with shortened windows. The pattern is consistent: foundation model providers optimise for upgrading their user base to current capability tiers, not for preserving the operational stability of the workflows their earlier models enabled. The difference here is that Codex CLI positions itself as infrastructure. Infrastructure tooling has different retirement expectations than chat interfaces.

## The Assistant Complaint

The most widely shared reaction to the retirement came from Vincent Schmalbach, whose post "OpenAI Is Retiring the Last Model That Behaves Like an Assistant" circulated through developer communities in the days before the deadline.[^6] The title captures something real.

GPT-5.5 had a characteristic that is harder to quantify than benchmark scores: it responded to corrective instructions well. When a Codex session produced output that drifted from intent, a brief natural-language correction reliably brought it back. The model seemed, in the vocabulary developers used in community threads, to be paying attention to what you actually wanted rather than to its training distribution.

This is not a capability you can find in an evaluation benchmark. SWE-Bench does not measure whether a model incorporates mid-session feedback gracefully. HumanEval does not capture conversational coherence across a multi-turn agentic loop. The qualities that made GPT-5.5 the model teams chose to leave running in overnight jobs — the ones that required minimal babysitting — are precisely the qualities that current benchmarks are not designed to surface.

GPT-6.1 Sol performs better on every published metric. It is genuinely faster. Its code generation quality is measurably higher on held-out evaluations. For greenfield tasks with well-defined specifications, it is the better tool. For the longer, messier, more ambiguous workflows that real engineering work produces — the ones where a model's ability to stay aligned with evolving intent matters as much as its raw output quality — the comparison is less clear.

## What This Means for Codex CLI Configuration

There is a practical path through this. It involves three decisions:

**Decide on a default model for your team.** GPT-6.1 Sol is the appropriate default for most interactive Codex sessions. The improvement in output quality justifies the cost increase for tasks where a developer is present to review and iterate. Set this in `config.yaml` at the repository root and let it propagate.

**Audit your automation for model pinning.** Any workflow that explicitly specifies `gpt-5.5` in a config file, an AGENTS.md, or a CI definition needs updating. The safest approach is to remove the explicit pin and let Codex use its configured default, so future model transitions require only one change. Where the automation runs at high frequency and cost matters, evaluate Terra before defaulting to Sol.

**Consider Terra for high-volume, low-complexity automation.** Scheduled dependency upgrades, automated test generation for new code, CI code-review comments on small PRs — these are tasks where the quality ceiling of a more capable model provides marginal return. Terra's cost profile makes sustained automation economically viable in a way Sol's does not.[^4]

The one caveat worth naming: API-key-authenticated Codex sessions may retain GPT-5.5 availability for a transitional period, based on community reports in the last 48 hours.[^3] OpenAI has not confirmed this officially. Do not design workflows around it. Plan the migration now and treat any continued access as a bonus rather than a dependency.

## The Structural Shift

I have been working in software long enough to recognise what a forced migration actually signals. It is not primarily a technical decision. It is a statement about who bears the cost of change.

When OpenAI retires a model, the cost of that retirement falls on the teams whose workflows depend on it. The engineering time spent updating configuration files, debugging failed CI jobs, re-evaluating model choices, and communicating the change to stakeholders — none of that appears in OpenAI's metrics. It appears in your sprint backlog.

The Stack Overflow 2026 Developer Survey showed Claude Code at 65.5% adoption against Codex CLI's 29.5%.[^7] Community threads in the days around this retirement cite the friction explicitly: the GPT-5.5 retirement is accelerating teams' evaluation of whether Codex CLI is the right long-term choice for their automation workflows. That is a significant response to a model retirement. It suggests the teams affected are not simply annoyed about a migration — they are questioning whether the platform's model lifecycle management is compatible with their operational needs.

I do not know whether that question will resolve in Codex's favour. The DevDay announcements — Codex Cloud, the /agents view, the Code Review integration — represent a genuine platform investment.[^8] The infrastructure is improving. The model lifecycle question is separate from the capability question, and it is the lifecycle question that the retirement has surfaced.

## What Comes Next

GPT-6.1 Sol Ultrafast, announced as part of the 28 Days of Shipping series, offers response latencies up to eight times faster than Sol standard at a pricing tier designed for high-frequency automation.[^9] If the cost profile proves competitive, it changes the economic argument for high-volume CI integration. That is worth watching in the next few weeks as independent benchmarks appear.

The /agents view (v0.156.0) and `instant_interrupt` steering (v0.159.0) change the ergonomics of multi-agent workflows in ways that are meaningfully better than what GPT-5.5 workflows were built around.[^10] There are real reasons to migrate forward, beyond compliance with OpenAI's retirement timeline.

The question worth sitting with is not whether GPT-6.1 Sol is a better model. It almost certainly is, on most dimensions. The question is whether a developer ecosystem built on automation and infrastructure can tolerate a model lifecycle that treats predictable, long-running workflows as a second-order concern.

The emoji in OpenAI's farewell post landed differently than intended.

---

[^1]: OpenAI, official statement and CLI guidance, October 14, 2026. Confirmed retirement scope: ChatGPT (all tiers), ChatGPT Work, Codex (UI + CLI plan-authenticated sessions).
[^2]: OpenAI X post, October 13, 2026: *"On October 14, it's time to say farewell to GPT-5.5 in ChatGPT, ChatGPT Work, and Codex across all plans. If you use GPT-5.5 in Codex, switch to GPT-6.1 Sol or GPT-6 Astra. Thanks for everything, 5.5 🫡"*
[^3]: Community reports, GitHub Discussions and r/ChatGPTCoding, October 12–14, 2026. API-key-authenticated session retention unconfirmed by OpenAI as of publication.
[^4]: OpenAI model documentation, October 2026. GPT-6.1 Sol: \$2/\$10 per million tokens (input/output). Terra documented as "competitive with GPT-5.5 at lower cost." See `https://developers.openai.com/codex/models` for current pricing.
[^5]: Community legacy-option request, filed September 15, 2026. Formally rejected by OpenAI. Thread activity surged in the 72 hours before retirement.
[^6]: Vincent Schmalbach, "OpenAI Is Retiring the Last Model That Behaves Like an Assistant," circulated via developer communities, October 2026.
[^7]: Stack Overflow 2026 Developer Survey, AI coding tool adoption: Claude Code 65.5%, Codex CLI 29.5%.
[^8]: OpenAI DevDay 2026, San Francisco, September 29, 2026. See companion article: *OpenAI DevDay 2026: Codex Cloud, the /agents View, GPT-6.1 Sol, Code Review, and Security Cloud*.
[^9]: OpenAI 28 Days of Shipping, Day 4: Instant steering + GPT-6.1 Sol Ultrafast (up to 8× faster than Sol standard). October 2026.
[^10]: Codex CLI release notes, September–October 2026. `/agents` view shipped in v0.156.0 (DevDay 2026 cluster); `instant_interrupt` shipped in v0.159.0 (September 29, 2026); managed worktrees introduced in v0.162.0 (October 8, 2026); OSC 7501 terminal status extended to all terminals in the v0.163.0-alpha sprint (October 2026). See companion article: *Codex v0.163.0: Managed Worktrees, OSC 7501, gRPC-stdio, and Agent UX Maturity*.
