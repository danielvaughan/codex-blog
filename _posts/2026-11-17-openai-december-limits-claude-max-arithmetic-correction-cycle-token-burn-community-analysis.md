---
title: "The December Limits Maths: How the Community Built an Analytical Case Against OpenAI's Quota Merger"
date: 2026-11-17T14:00:00+00:00
last_modified_at: 2026-11-25T11:40:36Z
parent: "Articles"
nav_order: 2118
tags:
  - codex-cli
  - openai
  - pricing
  - quota
  - rate-limits
  - developer-experience
  - claude-code
  - claude-max
  - pro-plan
  - billing
  - trust
copy_edited: true
rating: 4.5
depth: 4
originality: 5
accuracy: 4
substack: 5
---

# The December Limits Maths: How the Community Built an Analytical Case Against OpenAI's Quota Merger

The community protest over OpenAI's December 2026 quota merger has reached page 12 of the original thread. What started as an emotional response — *this feels unfair* — has matured into something more difficult to dismiss: a structured analytical case built from real usage data, arithmetic comparisons, and competitive benchmarking.

This article is not a recap of the original announcement. That is covered elsewhere. This is an account of what happened when developers stopped venting and started doing maths.

---

## How the Argument Changed on Page 12

The first ten pages of the thread were dominated by the two emotions you would expect: anger at the change, and anxiety about what it meant in practice. Page 12 introduced something different — a dissenter named Phivo who made a substantive pro-consolidation argument, and a majority response that accepted the argument's premise while rejecting its conclusion.

Phivo's position: OpenAI faces genuine and sustained cost pressure. The separate allowances for ChatGPT and Codex were always, in economic terms, a cross-subsidy — conversational usage funded agentic usage, or vice versa, depending on whose margin you were looking at. No commercial organisation can sustain indefinite cross-subsidies. The merger is not an act of bad faith; it is a delayed correction.

The community's response to this was notable for its quality. The majority did not dispute the cost pressure argument. They accepted it. The objection crystallised into a single, precise demand: *the cost argument is acceptable; the opacity is not*.

The phrasing that received the most agreement across page 12 was this:

> "If OpenAI will not publish the numbers, the only rational interpretation is that the numbers are unfavourable to us."

This is a different kind of objection than the first ten pages produced. It is not emotional. It is an argument from information theory: when a party to an economic exchange withholds data that would allow the other party to evaluate whether the exchange is fair, that withholding is itself evidence about the direction of unfairness. The community has stopped asking OpenAI to reverse the decision. They are now asking OpenAI to prove, with figures, that the unified allowance is not a cut disguised as a consolidation.

---

## The Correction-Cycle Multiplier: A New Billing Mechanism Nobody Planned For

The most technically novel contribution on page 12 is the identification of what several developers have started calling the **correction-cycle token burn**.

The mechanism works like this. AI models, particularly large reasoning models operating on long-horizon agentic tasks, periodically lose coherent context within a single conversation. This is not a hallucination in the classical sense — the model does not generate false information — but a context-window management failure in which the model begins re-deriving information it has already established, re-issuing tool calls that have already been completed, or requesting user clarification on decisions that were made ten turns earlier.

Each correction cycle generates tokens. Under the current separate-limits architecture, those tokens come from the Codex allowance, which was sized (however imperfectly) around agentic task consumption patterns. Under the unified allowance, they come from the same pool as a morning's conversational queries.

The community's concern is not that correction cycles will become more frequent — the merge does not change the underlying model behaviour. The concern is that the unified allowance will make correction-cycle waste immediately visible and painful in a way that the current architecture conceals. When a 90-minute refactor burns 15,000 unexpected tokens because the model lost context and ran the same grep sequence three times, that waste currently disappears into a separate Codex bucket. From December, it disappears into the same bucket as the rest of the week.

This is, in structural terms, a billing multiplier. Correction cycles that were previously invisible will now compete directly with conversational usage for headroom. The developers most affected are those running the longest tasks — which is to say, the developers for whom Codex CLI is most central to their workflow.

---

## Claude Max: When Competitor Comparisons Became Arithmetic

The competitive dimension of this thread has been present since page one, but it has undergone a qualitative change. Through the first ten pages, the references to Claude Code and Claude Max were rhetorical: *Claude doesn't do this*, *I might switch*. On page 12, the references became numerical.

A developer identified as running a typical senior engineering workflow — approximately 40 hours per week of AI-assisted development — published a side-by-side comparison. The methodology was straightforward: take a representative two-week period of Codex CLI usage logs, extract the token volumes by task type, apply the published Claude Max pricing structure, and calculate whether the unified Codex allowance (assumed to be equivalent to the current aggregate) covers the same workflow.

The conclusion, which received significant agreement from developers posting similar calculations: for heavy agentic workflows, **Claude Max's separate conversational and agentic allowances produce materially better effective capacity than the projected unified OpenAI allowance at equivalent price points**.

The important word in that sentence is *projected*. No unified allowance figure has been published. The comparison depends on assumptions about what the merged allowance will actually contain — and those assumptions, the community notes, can only be generous or pessimistic in the absence of real data. The developers doing this arithmetic are assuming a pessimistic unified allowance, because the alternative requires trusting an organisation that has not yet earned that trust on this specific question.

This matters because the nature of a threat to switch has changed. A rhetorical threat to switch is a negotiating move. An arithmetic case for switching is a migration plan. The community is building migration plans.

---

## What OpenAI Has Not Done — and When It Would Need To

The specific demands from the community as of page 12 are concrete and time-bounded:

**Minimum viable disclosure before December:**
- Token consumption rates per model per task type, specifically distinguishing GPT-6.1 Sol interactive versus GPT-6.1 Sol agentic
- The unified allowance figure for each subscription tier
- A real-time consumption dashboard or, at minimum, a weekly usage summary email

**Additional requests (secondary priority):**
- An explanation of why the correction-cycle token-burn issue was not considered in the allowance design
- A commitment to a review period — 60 or 90 days post-merger — with a published mechanism for allowance adjustment based on observed consumption data

None of these have been addressed. No OpenAI community manager has posted in the thread since page three. This silence is now itself a data point. The developers who were still giving OpenAI the benefit of the doubt on page five are, by page 12, explicitly referencing the silence as evidence for the pessimistic interpretation.

---

## The Structural Argument OpenAI Has Not Made

There is a version of this announcement that would have landed better. It goes roughly like this:

*We are merging the allowances because separate limits create an artificial separation between workloads that, at the infrastructure level, do not behave differently. A token is a token. We are not reducing total allowance — we are removing an accounting fiction that made some users feel they had more capacity than they actually had. Here are the unified figures. Here is a calculator. We expect most users to find that their workflows are unaffected.*

That version requires publishing the numbers. If the numbers are genuinely neutral or favourable, publishing them costs nothing and wins back significant trust. If the numbers are unfavourable, publishing them is commercially painful — but the alternative is the community assuming the worst and building migration plans accordingly.

The community on page 12 is not operating on sentiment. It is operating on the absence of data, which is itself data.

---

## What This Means Going Into December

The December 2026 usage-limits merger is now a trust event as much as a billing event. The question it poses is not primarily *will my workflow still work* but *can I still plan around this platform*.

Developers make long-horizon commitments to development tools. They build workflows, write documentation, establish team practices, and invest in tooling integrations. Those commitments require a belief that the platform's economics are knowable and stable. The merger announcement, combined with the ongoing silence, is damaging that belief in a way that a price increase communicated transparently would not.

The community's arithmetic comparisons with Claude Max are not principally about Claude Max. They are a signal that developers have started doing the work of evaluating alternatives — not because the alternatives are necessarily better, but because the information required to trust the current platform has not been provided.

If OpenAI publishes the unified allowance figures before December, the analytical case for switching weakens considerably. The numbers may be fine. If they are fine, publishing them is free. The fact that they have not been published is what the community on page 12 has decided it cannot stop thinking about.

---

*Source: community.openai.com/t/please-keep-chatgpt-and-codex-usage-limits-separate-in-december-2026/1411466 (pages 11–12, 2026-11-17); reddit-sentiment/2026-11-17-daily-report.md. See also: [OpenAI's December Quota Merger: What Happens When Your Chat and Code Share the Same Allowance](/articles/2026-11-16-openai-december-usage-limits-merge-chatgpt-codex-shared-allowance-developer-impact/) (Nov 16 background article).*
