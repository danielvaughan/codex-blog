---
title: "28 Days of Shipping: The Final Scorecard"
date: 2026-11-01T08:00:00+00:00
last_modified_at: 2026-11-05T11:07:39Z
parent: "Articles"
nav_order: 1211
tags:
  - codex-cli
  - openai
  - 28-days-shipping
  - community
  - analysis
  - product-strategy
  - sprint-retrospective
copy_edited: true
---

# 28 Days of Shipping: The Final Scorecard

![Sketchnote diagram for: 28 Days of Shipping: The Final Scorecard](/sketchnotes/articles/2026-11-01-28-days-of-shipping-end-of-sprint-scorecard-openai-codex.png)

**Sprint window:** 5 October – 1 November 2026 (28 days)
**Final data through:** Day 27 (31 October 2026)
**Companion piece:** [28 Days of Shipping: What the Sprint Scorecard Tells Us About OpenAI's Priorities](/2026/10/22/28-days-sprint-scorecard-what-the-data-tells-us-openai-codex/) (mid-sprint analysis, Day 17)

---

The sprint is over. Twenty-eight days of announced daily deliveries — features, resets, and a few surprises — have run their course. On 22 October, the [mid-sprint analysis](/2026/10/22/28-days-sprint-scorecard-what-the-data-tells-us-openai-codex/) assessed the first seventeen days, identified the community's Tier 1 wishlist, and set out what Days 18–28 needed to deliver to call the sprint a success. This is the final reckoning.

The short version: OpenAI shipped more Codex CLI substance in the sprint's second half than in its first. Three stable releases crossed the line — v0.164.0, v0.165.0, and v0.166.0 — each carrying meaningful features. The enterprise story improved dramatically. The competitive positioning against Claude Code is stronger than it was on Day 1. And yet the sprint closed with a self-inflicted crisis — a Pro allowance cut that lasted under 24 hours before being reversed — that crystallised the community's most enduring concern about OpenAI: that the operational layer is still being managed reactively.

---

## The Full Sprint Scorecard

### Days 1–17: The First Half (Recap)

The mid-sprint scorecard covered these in detail. The headline figures: four of the first seventeen days were usage resets rather than feature deliveries; the best-received features were infrastructure improvements (12-hour prompt caching at 91%, context window expansion at 85%); and the worst-received feature was "edit previous messages" in the desktop app (8%) — a UX nicety delivered into a community asking for quota, reliability, and cost improvements.

The first half established a pattern: infrastructure features resonate, surface-level UX features do not, and resets are tolerated but consume sprint days that the community would rather see spent on features.

### Days 18–28: The Second Half

| Day | Date | Feature / Event | Signal |
|-----|------|-----------------|--------|
| 18 | 22 Oct | Streaming interrupt fix; v0.163 backports | Positive — patch responsiveness |
| 19 | 23 Oct | **v0.164.0 stable: /presets, image previews, Enterprise MCP GA, Bedrock default-on** | Positive |
| 20 | 24 Oct | Agent wake orchestration primitive | Positive (enterprise) |
| 21 | 25 Oct | Credential masking, remote execution hardening | Positive (security) |
| 22 | 26 Oct | **v0.165.0 alpha opens: observability sprint begins** | Positive |
| 23 | 27 Oct | **v0.165.0 stable: GCP Vertex AI built-in, GPT-6.1 Terra, Memory v2 default** | Strongly positive |
| 24 | 28 Oct | OpenTelemetry GA, PEM client identities, `codex doctor --network-only` | Positive (enterprise) |
| 25 | 29 Oct | **v0.166.0 stable: `codex queue --resume`, keyboard transcript nav, Bazel/Windows V8 fix** | Positive |
| 26 | 30 Oct | **Pro allowance cut announced** | Strongly negative — community crisis |
| 27 | 31 Oct | **OpenAI reverses Pro cut; Pro 100/500 allowances doubled** | Mixed — relief with lasting distrust |
| 28 | 1 Nov | Sprint closes | — |

The second half delivered three stable releases in ten days — v0.164.0 on 23 October, v0.165.0 on 27 October, and v0.166.0 on 29 October. That cadence (a stable release roughly every four days) is faster than at any point since the sprint began.

---

## What the Second Half Actually Delivered

### v0.164.0: Prompt Management and Enterprise Graduation

v0.164.0 is the sprint's "broad surface area" release. `/presets` — ten numbered slots for reusable prompts and slash commands — addressed one of the most practical daily-workflow friction points for power users. Inline `/model` and `/permissions` commands removed a two-step interaction. Image previews arrived.

The more significant story in v0.164.0 is under the hood: Enterprise MCP authentication reached general availability, Bedrock setup defaulted on, and a credential masking layer shipped for remote execution environments. The sprint's enterprise thread — which began with MCP OAuth default-on and zero-trust agent runtime in the alpha sprints — culminated in a release that is genuinely ready for procurement conversations.

### v0.165.0: The Multi-Cloud and Observability Release

v0.165.0 is the sprint's most strategically significant release. The headline is the built-in Google Cloud Vertex AI provider — the first first-party cloud integration beyond OpenAI-hosted endpoints. Enterprise teams with existing GCP infrastructure, data residency requirements, or committed cloud spend can now route Codex CLI through their own Vertex AI deployments. This directly addresses the model lock-in complaint that the Stack Overflow 2026 Developer Survey named as a Codex structural disadvantage.

GPT-6.1 Terra arrived in the model catalogue in v0.165.0. Its relevance goes beyond a model upgrade: Terra carries improved instruction-following relative to Sol, making it the recommended mitigation for the GPT-6.1 Sol AGENTS.md compliance regression that has been open for 23 days as of the sprint close. OpenAI has not acknowledged the Sol regression publicly, but shipping Terra and documenting it as the recommended model for compliance-sensitive workflows is a de facto acknowledgement.

Memory v2 becoming the new default for new installs is the sleeper change of the sprint. New users now experience Codex as a tool that remembers their preferences and specifications across sessions. This is the behaviour that the community has long associated with Claude Code's `CLAUDE.md` system. The comparison is now less stark.

The observability layer that graduated in v0.165.0 — turn-level tracing, streaming instrumentation, tool-dispatch logging, Guardian v2 — matters primarily to enterprise users, but it is the kind of capability that unlocks production deployments. You cannot run Codex CLI in a regulated enterprise environment without audit trails. v0.165.0 crossed that threshold.

### v0.166.0: Async, Terminal, and Windows

v0.166.0 delivered `codex queue --resume` — the ability to list and resume all paused or queued threads not currently in memory with a single command. This is the first genuinely async session management primitive in Codex CLI. For teams running overnight batch jobs, CI-triggered analysis runs, or review pipelines, the ability to queue multiple tasks, close the terminal, and resume everything the next morning with a single command is a meaningful workflow change.

Keyboard-driven transcript selection completed the terminal-native UX story: word, line, and page navigation via `w`, `b`, `j`, `k`, `PgDn`, `PgUp`. For developers in tmux, Zellij, or SSH sessions, mouse selection is unreliable or unavailable. Full keyboard navigation is not a cosmetic improvement.

The Bazel build migration to source-built V8 with native Windows support addressed what had become a systemic embarrassment: three consecutive major releases shipped Windows regressions. v0.164.0 shipped a Windows file deletion P0 and desktop crash regression. v0.165.0 shipped a Windows VS Code 1.143 regression. v0.166.0 shipped a VS Code 1.143.1 launch regression on day one of stable, driven by the Electron version mismatch that the Bazel migration was designed to close. The fix — a structural migration, not another patch — suggests the regression cycle is now closed. One release is not enough to confirm that, but the architectural change is correct.

---

## Did the Second Half Deliver the Community's Tier 1 Wishlist?

The [mid-sprint analysis](/2026/10/22/28-days-sprint-scorecard-what-the-data-tells-us-openai-codex/) identified four Tier 1 priorities from community data. This is how each landed.

### 1. Remote WSL Control (18 upvotes, most-upvoted open Ideas request)

**Not delivered.**

This was the single feature most explicitly named as a competitive signal against Claude Code. It did not ship in Days 18–28. The Ideas forum request remains open. The sprint closes without addressing it.

### 2. Quota Increase for Pro/Team Tiers

**Partially delivered — but badly handled.**

The sprint delivered Pro 100 and Pro 500 allowance doublings as part of the Day 27 reversal announcement. Those are genuine improvements. But the manner of delivery — cutting Pro 200 allowances first, generating a community crisis, then reversing under pressure while doubling the other tiers — is not how a quota improvement lands well. The community received the outcome it wanted, but through a process that damaged trust rather than building it.

The underlying structural problem — published, explicit numerical limits for each tier — remains unresolved. The community is still guessing at precise limits.

### 3. Cost Reduction for API Users

**Not meaningfully delivered.**

No per-token pricing reduction or new efficiency tier shipped in Days 18–28. The Vertex AI integration creates an indirect path to cost control for GCP-committed enterprises, but it is not a direct response to the API pricing requests.

### 4. Undo Command (466 upvotes)

**Not delivered.**

The 466-upvote agentic undo request — the ability to reverse the last agent action cleanly — did not ship. This remains the most-upvoted unaddressed feature request in the Ideas forum.

### Summary: 0 of 4 Tier 1 Items Fully Delivered

The Tier 1 wishlist was zero-for-four. The sprint's second half delivered significant value — but along a different axis than the community's stated operational priorities. The features that shipped were primarily developer experience improvements (presets, keyboard navigation, async session management) and enterprise enablement (Vertex AI, MCP GA, OTel). The community's operational concerns — remote WSL, cost, quota transparency, undo — went unaddressed.

---

## The Pro Allowance Crisis: The Sprint's Defining Event

On Day 26 (30 October), OpenAI announced a reduction in Codex Pro 200 usage limits. Within 24 hours, the announcement was reversed: Pro 200 limits would not be reduced, and Pro 100 and Pro 500 allowances would be doubled.

The reversal was the right call. But the sequence — announcement, backlash, reversal — is now the third such cycle in 2026 by the community's count. The mechanism is visible and repeating: OpenAI tests a limit change, the community objects loudly, OpenAI reverses. What is absent is any public communication about what caused the proposed cut, what evidence led to the reversal, or what guardrails exist to prevent the cycle from repeating.

The most consequential element is not the reversal itself but the cohort that completed their migration to Claude Code during the 24-hour crisis window. Community threads on 31 October are explicit about this: developers who moved during the announcement are not coming back. The reversal recaptured the holdouts but not the early movers. The trust damage is asymmetric — it is permanent for the segment that left, temporary for the segment that stayed.

The community's articulation of the structural problem is precise: it is not about limits per se, but about predictability. A well-communicated limit that users can plan around is manageable. An unannounced cut followed by a rapid reversal is unpredictable, and unpredictability is fatal to enterprise planning cycles.

---

## The Three Windows Regressions

The sprint's most under-reported narrative is the Windows regression pattern. Three consecutive major stable releases — v0.164.0, v0.165.0, and v0.166.0 — each shipped with a Windows platform regression. The v0.166.0 regression (VS Code 1.143.1 launch failure) was the most visible, generating 42 comments and 39 upvotes on the issue tracker within the first week of stable.

The Bazel migration in v0.166.0 is the structural fix: building V8 from source removes the dependency on pre-built Electron binaries that caused the mismatch chain. If the migration holds, the regression cycle closes here. If it does not — if v0.167.0 ships another Windows regression — the community's "pattern, not a coincidence" framing will harden.

For Windows developers, the sprint delivered the fix one would have hoped to see by Day 7.

---

## The GPT-6.1 Sol Regression: Still Open

The sprint opens and closes with the GPT-6.1 Sol AGENTS.md compliance regression unresolved. The model acknowledges write constraints in `AGENTS.md`, then re-violates them one to three turns later. The regression has been documented across 23 days as of 31 October. OpenAI has not acknowledged it publicly.

The practical mitigation — switching to GPT-6.1 Terra — shipped in v0.165.0 and is now accessible in the model catalogue. The [Terra escape hatch article](/2026/10/31/gpt61-terra-escape-hatch-agents-md-compliance-migration-sol-regression/) documents the full migration path. But the Sol regression remains live for any team still using it, and the absence of an official acknowledgement means enterprise teams cannot formally close the issue in their risk registers.

---

## What the Sprint's Second Half Changed Competitively

The Stack Overflow 2026 Developer Survey entering the sprint showed Claude Code at 65.5% developer mind-share versus Codex at 29.5%. The sprint did not close that gap — a 28-day initiative was never going to move a structural survey result. But it shifted the qualitative picture in three areas.

**Multi-cloud:** Vertex AI built-in is the sprint's most strategically durable change. Six months ago, routing Codex through GCP required custom provider hacks. It is now a first-party configuration option. Enterprise teams with GCP infrastructure now have a supported path to Codex CLI without OpenAI model lock-in. This is not a feature that shows up in developer mind-share surveys immediately. It shows up in Q1 2027 enterprise procurement cycles.

**Observability:** Guardian v2 and the OTel graduation in v0.165.0 crossed a production deployment threshold. The teams building compliance-grade agentic workflows have a path forward. Claude Code's local-first model still has advantages in regulated environments, but the Codex CLI observability story is no longer a gap; it is a differentiator.

**Memory:** Memory v2 as default closes the most-visible UX gap against Claude Code's `CLAUDE.md` system for new users. This matters for first impressions and for developer advocates comparing tools.

The remote WSL gap, the undo gap, and the cost transparency gap remain.

---

## The Sprint's Final Verdict

**What the sprint was:** An enterprise hardening sprint with a consumer framing. The CLI releases — four stables in 28 days, covering multi-cloud, observability, async session management, and terminal UX — are substantive and durable. The sprint did real work.

**What the sprint was not:** A response to the community's stated operational priorities. Remote WSL, undo, cost reduction, quota transparency — zero for four on the Tier 1 wishlist. The features that shipped addressed enterprise enablement and developer experience; the features the community asked for most loudly were not on the sprint plan.

**The sprint's legacy:** The Pro allowance crisis will define how this sprint is remembered, regardless of the CLI work. The reversal was correct. The cycle — cut, backlash, reverse — is the problem. The community's trust in OpenAI's pricing decisions has a shorter half-life now than it had on Day 1. That is not a feature gap. It is harder to close than a feature gap.

**The competitive picture:** Better than on Day 1. The Vertex AI integration, Memory v2 default, and OTel GA are durable improvements. The mind-share survey gap will not move in 28 days. The enterprise procurement pipeline is a different story.

The sprint ends. The product work continues.

---

## Open Items Carried Forward

| Item | Status | Notes |
|------|--------|-------|
| Remote WSL control | Open | Most-upvoted Ideas request; not addressed in sprint |
| Undo command | Open | 466 upvotes; not addressed |
| Explicit per-tier usage limits (published) | Open | Core trust-repair action post-crisis |
| GPT-6.1 Sol AGENTS.md regression | Open | Terra mitigation documented; Sol unacknowledged |
| Windows regression pattern | Structurally fixed (v0.166) | Bazel migration shipped; monitor v0.167 |
| API token spike (mulderpf) | Status unclear | No official response found as of sprint close |

---


[^1]: Mid-sprint scorecard (Day 17): [28 Days of Shipping: What the Sprint Scorecard Tells Us](/2026/10/22/28-days-sprint-scorecard-what-the-data-tells-us-openai-codex/). 22 October 2026.
[^2]: v0.164.0 stable: [Codex CLI v0.164.0 Stable: /presets, Image Previews, Enterprise MCP GA, and Bedrock Default-On](/2026/10/23/codex-v0164-stable-release-presets-image-previews-enterprise-mcp-bedrock/). 23 October 2026.
[^3]: v0.165.0 stable: [Codex CLI v0.165.0 Stable: GCP Vertex AI Built-In, GPT-6.1 Terra, Memory v2 Default](/2026/10/28/codex-v0165-stable-gcp-vertex-gpt61-terra-memory-v2/). 28 October 2026.
[^4]: v0.166.0 stable: [Codex CLI v0.166.0 Stable: codex queue --resume, Keyboard Transcript Navigation, and GPT-6.1 Terra](/2026/10/31/codex-v0166-stable-queue-resume-keyboard-transcript-terra/). 31 October 2026.
[^5]: GPT-6.1 Sol regression: [GPT-6.1 Sol AGENTS.md Compliance Regression](/2026/10/26/gpt61-sol-agents-md-compliance-regression-acknowledge-apologise-re-violate/). 26 October 2026.
[^6]: Terra escape hatch: [GPT-6.1 Terra as the Escape Hatch: AGENTS.md Compliance Migration Guide](/2026/10/31/gpt61-terra-escape-hatch-agents-md-compliance-migration-sol-regression/). 31 October 2026.
[^7]: Pro allowance cut and reversal: [OpenAI Pro Codex Allowance Cut: Migration Signal](/2026/10/31/openai-pro-codex-allowance-cut-migration-signal/). 31 October 2026.
[^8]: Reddit Sentiment Monitor — 31 October 2026. codex-resources/reddit-sentiment/2026-10-31-weekly-report.md.
[^9]: Reddit Sentiment Monitor — 29 October 2026. codex-resources/reddit-sentiment/2026-10-29-weekly-report.md.
[^10]: Stack Overflow Developer Survey 2026: Claude Code 65.5% vs Codex 29.5% developer mind-share.
[^11]: ikaijua/Awesome-AITools sprint tracker: community approval data, Days 1–17, October 2026.
[^12]: OpenAI Ideas forum: Remote WSL control request, Undo command request. community.openai.com. October 2026.
