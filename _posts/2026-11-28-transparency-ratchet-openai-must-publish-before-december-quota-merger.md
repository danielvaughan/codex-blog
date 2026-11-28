---
title: "The Transparency Ratchet: What OpenAI Must Publish Before the December Quota Merger"
date: 2026-11-28T09:00:00+00:00
last_modified_at: 2026-11-28T13:07:40Z
parent: "Articles"
nav_order: 2133
tags:
  - codex-cli
  - openai
  - pricing
  - quota
  - rate-limits
  - developer-experience
  - billing
  - trust
  - pro-plan
  - governance
copy_edited: true
---

# The Transparency Ratchet: What OpenAI Must Publish Before the December Quota Merger

December is three days away. The OpenAI developer forum thread protesting the December quota merger has run to fourteen pages. No OpenAI staff member has posted since page three. No unified allowance figures have been published. No consumption dashboard exists. No model-level breakdown has been offered.

The window to act credibly is closing. This article is about what that window requires — and why closing it without acting has a cost that compounds.

---

## The Ratchet Mechanism

The phrase *transparency ratchet* describes a governance dynamic in which each disclosure failure makes the next one more expensive to fix. It works like this:

When OpenAI first announced the December quota merger in mid-November, the community response was strong but not conclusive. Developers were angry, but the dominant interpretation was charitable: OpenAI had made a communication error, would publish supporting figures, and the anger would be proportionate to whatever the figures showed.

Eleven days passed. No figures. The community response shifted from *we expect numbers* to *the absence of numbers is itself a signal*. The phrasing that accumulated the most agreement across page twelve of the thread was precise: *"If OpenAI will not publish the numbers, the only rational interpretation is that the numbers are unfavourable to us."*

This is not an emotional argument. It is an argument from information theory. When a party to an economic exchange withholds data that would allow the other party to evaluate whether the exchange is fair, that withholding is itself evidence about the direction of unfairness. The community did not need to assume bad faith to arrive at that conclusion. They simply needed to apply the logic.

Each additional day of silence turns the ratchet one notch further. The numbers that would have been received as routine disclosure on 17 November will now be received as a response to a governance crisis. The figures have not changed; what has changed is the interpretive frame into which they will land. That frame hardens a little more each day.

November 28. Three days to December.

---

## What the Community Is Actually Asking For

The demands from the developer community as of this writing are specific, bounded, and not unreasonable. They are not a demand to reverse the merger. They are a minimum viable disclosure checklist.

**Token consumption rates per model per task type.** The specific ask is a distinction between GPT-6.2 Sol interactive and GPT-6.2 Sol agentic — the same model behaves differently in a conversational session than in a multi-hour refactor task. Developers building their own cost projections cannot do the arithmetic without this input. The data exists in OpenAI's billing infrastructure. The barrier to publishing it is not technical.

**The unified allowance figure for each subscription tier.** This is the most basic ask. Under the new structure, a Plus subscriber will have a single token-based allowance that covers both ChatGPT and Codex. What is that allowance? No figure has been published. The community cannot evaluate whether the unified allowance represents equivalent capacity, reduced capacity, or improved capacity without a number. All projections are therefore pessimistic by default, because a pessimistic assumption is the only defensible one when the data is withheld.

**A real-time consumption dashboard.** Not a percentage indicator. An absolute counter, broken down by model, distinguishing interactive from agentic consumption. The rate-limit megathread — 1,048 replies as of its peak, the most active Codex thread in the community's history — exists in large part because developers have been aggregating consumption data via community posts because the platform does not expose it directly. A session freeze silently burning 30 per cent of a weekly Pro allowance with no output is not a hypothetical; it is a documented incident. Developers cannot manage what they cannot measure.

**A post-merger review commitment.** The specific ask is a 60-to-90-day review window with a published mechanism for allowance adjustment based on observed consumption data. This is the professional standard for significant pricing changes in developer tool subscriptions. It converts the merger from a unilateral decision into a monitored transition.

None of these asks require reversing the decision. They require treating developers as customers capable of evaluating it.

---

## Why This Is Harder Than It Looks

There is a structural reason the correction-cycle token burn makes this disclosure more urgent than a simple allowance question.

The compaction spiral — documented in Issues [#60538](https://github.com/openai/codex/issues/60538) and [#60791](https://github.com/openai/codex/issues/60791) — is a feedback loop in which Codex CLI's context compaction system repeatedly rewrites the conversation history, invalidating the cache prefix, triggering full-price re-ingestion on each turn, and driving context back toward the limit that triggers the next compaction event. One session documented 72 compaction events with cache reuse falling to 15.7 per cent. A developer lost a full weekly Pro allowance — \$200 worth — in eight hours.

The correction-cycle burn is a related but distinct failure mode. Large reasoning models periodically lose coherent context within a session — not hallucinations, but context-window management failures where the model re-derives established information, re-issues completed tool calls, or requests clarification on decisions made ten turns earlier. Each correction cycle generates tokens. Under the current separate-allowance architecture, those tokens come from the Codex bucket, which was sized around agentic task consumption patterns. Under the unified allowance, they come from the same pool as a morning's conversational queries.

The community's concern is not that correction cycles will increase in frequency. It is that unified limits will make correction-cycle waste immediately painful in a way that the current architecture conceals. When a 90-minute refactor burns 15,000 unexpected tokens because the model lost context and ran the same grep sequence three times, that waste currently disappears into a separate Codex bucket. From December, it disappears into the same bucket as the rest of the week.

This is a billing multiplier effect. It is not visible in any aggregate allowance figure. It requires model-level and task-type-level consumption data to reason about. This is precisely the data the community is asking for. The ask is not arbitrary — it is the minimum information set required to evaluate what a unified allowance actually means in practice.

---

## The Competitive Arithmetic Is Already Being Written

Through the first ten pages of the forum thread, references to Claude Code and Claude Max were rhetorical: *Claude doesn't do this*, *I might switch*. By page twelve, the references had become numerical.

A developer published a side-by-side comparison: two weeks of Codex CLI usage logs, token volumes by task type, applied against published Claude Max pricing. The conclusion — that Claude Max's separate conversational and agentic allowances produce materially better effective capacity than the projected unified OpenAI allowance at equivalent price points — spread across the thread and was confirmed by multiple developers posting similar calculations.

The important qualifier is *projected*. No unified allowance figure has been published. The comparison depends on assumptions, and in the absence of data, those assumptions are pessimistic. That is not irrational — it is the only defensible posture.

The nature of a threat to switch has changed. A rhetorical threat to switch is a negotiating move. An arithmetic case for switching is a migration plan. The community is building migration plans. The migration friction is real but finite. Developers have tooling, workflows, and muscle memory invested in Codex CLI. That investment does not evaporate at a forum post, but it does erode with each turn of the transparency ratchet.

---

## The Version of This Announcement That Would Have Landed Better

There is a credible alternative to silence. It goes roughly like this:

*We are merging the allowances because separate limits create an accounting separation that does not reflect how the underlying infrastructure works. A token is a token. We are not reducing total allowance — we are removing an accounting fiction. Here are the unified figures by tier. Here is a consumption calculator. We expect most users running standard workflows to find the unified allowance equivalent to or better than their current separate limits. For heavy agentic users, here is the per-model breakdown so you can evaluate your specific workflow. We are committing to a 90-day review.*

That announcement does not exist. The announcement that does exist is a structural change to how developers purchase and consume access, with no supporting data, landing on top of three weeks of active quota anxiety — the token spiral, the compaction spiral, and reports of unannounced capacity reductions on the Pro tier.

A pricing change announced without data, in that environment, is not received as a product decision. It is received as a pattern.

---

## The Disclosure Checklist

This is what OpenAI needs to publish before December 1 to change how the December merger lands. It is not a complete transparency wishlist. It is the minimum viable set.

| Disclosure | Why It Is Required |
|---|---|
| Unified allowance per tier (absolute units) | Developers cannot evaluate the change without a number |
| Token consumption rate: GPT-6.2 Sol interactive vs agentic | Required to project per-workflow impact |
| Correction-cycle token burn acknowledgement | Required to explain why agentic allowances cannot be sized like conversational allowances |
| Cache hit rate by task type | Required to understand real effective cost vs nominal token cost |
| 90-day post-merger review commitment | Professional standard for significant pricing changes |

Publishing this checklist does not guarantee a good outcome. The numbers may confirm the pessimistic interpretation. But publishing them while the numbers are what they are is recoverable. Not publishing them until the community has fully committed to migration plans is a different problem.

---

## What Happens If the Window Closes

The developer community's trust in a commercial platform is not restored in a single announcement. It accumulates across many interactions, mostly small. It erodes the same way.

The OpenAI developer community has an unusually long institutional memory. The October reversal of the Pro limit cut — when sustained community pressure produced a documented policy reversal — is cited explicitly in the current thread as evidence that organised pressure works. The community knows the precedent. It is applying it.

The difference between then and now is that the December deadline is external. OpenAI reversed the Pro limit cut after the community organised. It can publish the December merger figures before December. The first reversal took days. The second requires publishing data that already exists.

Three days. The ratchet is at the last notch before December.

---

*Sources: OpenAI Developer Community forum, community.openai.com/c/codex (November 2026, pages 1–14 of quota merger thread); The December Limits Maths (KB, 17 November 2026); OpenAI's December Quota Merger (KB, 16 November 2026); The Compaction Spiral (KB, 14 November 2026); The Token Spiral (KB, 10 November 2026); 2026-11-23 Weekly KB Update (KB, 23 November 2026).*
