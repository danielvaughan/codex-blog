---
title: "28 Days of Shipping: What the Sprint Scorecard Tells Us About OpenAI's Priorities"
date: 2026-10-22T09:00:00+00:00
last_modified_at: 2026-10-23T11:06:20+01:00
parent: "Articles"
nav_order: 1196
tags:
  - codex-cli
  - openai
  - 28-days-shipping
  - community
  - analysis
  - product-strategy
copy_edited: true
---

# 28 Days of Shipping: What the Sprint Scorecard Tells Us About OpenAI's Priorities

![Sketchnote diagram for: 28 Days of Shipping: What the Sprint Scorecard Tells Us About OpenAI's Priorities](/sketchnotes/articles/2026-10-22-28-days-sprint-scorecard-what-the-data-tells-us-openai-codex.png)

**Sprint window:** 5 October – 1 November 2026 (28 days)
**Data through:** Day 17 (21 October 2026)
**Analysis type:** Community approval data + feature classification pattern

---

OpenAI's "28 Days of Shipping" sprint is at its midpoint. Seventeen of twenty-eight days have results. The data tells a clear story — one that OpenAI may or may not be reading in the same way the community is.

This article examines the sprint scorecard through Day 17: what shipped, how it was received, what the approval patterns reveal about community priorities, and what the remaining eleven days need to deliver.

---

## The Scorecard: Days 1–17

| Day | Date | Feature | Community Approval |
|-----|------|---------|-------------------|
| 1 | 5 Oct | ~50% faster default generation speed | ~50% |
| 2 | 6 Oct | Free auto-review; simplified subscription tiers | **Reset** |
| 3–6 | 7–10 Oct | (Not publicly confirmed / sub-features) | — |
| 7 | 11 Oct | Context window 272k → 500k tokens | ~85% |
| 11 | 15 Oct | Fewer "at capacity" errors | ~83% |
| 12 | 16 Oct | Context reuse and prompt caching | 83% |
| 13 | 17 Oct | Usage reset event | **Reset** |
| 14 | 18 Oct | Edit previous messages in desktop thread | **8%** |
| 15 | 19 Oct | Global usage reset | **Reset** |
| 16 | 20 Oct | 12-hour prompt caching | **91%** |
| 17 | 21 Oct | Usage reset announced | **Reset** |

*Approval figures sourced from the ikaijua/Awesome-AITools community tracker.*

---

## The Pattern Is Unmistakable

The data is not ambiguous. Sort the days into two categories and the picture is stark:

**Efficiency/quota/cost/infrastructure days:**

| Day | Feature | Approval |
|-----|---------|----------|
| 7 | Context window 272k → 500k | 85% |
| 11 | Fewer at-capacity errors | 83% |
| 12 | Context reuse/caching | 83% |
| 16 | 12-hour prompt caching | **91%** |

Average: **85.5%**

**Interface/UX/feature days:**

| Day | Feature | Approval |
|-----|---------|----------|
| 1 | 50% faster speed | ~50% |
| 14 | Edit previous messages | **8%** |

Average: **~29%**

**Reset days:** 4 of 17 days (Days 2, 13, 15, 17)

The community is expressing a clear preference: they want Codex to cost less, use their quota more efficiently, and be reliably available. They are indifferent or actively negative about interface polish and UX features.

This is not a subtle signal. It is a statistically clean pattern across the data available.

---

## Why the Community Scores UX Features So Harshly

Day 14's score of 8% for "edit previous messages" is worth examining in detail, because it was not a bad feature. The ability to edit a previous message in a desktop thread is genuinely useful. It is the kind of QoL improvement that ships frequently and unremarkably in consumer applications.

The problem is context. The community evaluating this feature is not looking for a better chat interface. It is looking for answers to pressing operational questions:

- Why do I run out of quota mid-afternoon?
- Why is the API returning 2.5 billion tokens when I expected 350 million?
- Why does mid-response steering terminate my stream instead of steering it?

Against those questions, the ability to edit a previous message is not irrelevant — it simply signals that OpenAI is spending sprint days on the wrong layer of the product.

The 8% score is not a community saying "this feature is bad." It is a community saying "this feature is not what we need right now."

---

## The Reset Problem

Four of seventeen days — nearly a quarter of the sprint — have been usage resets rather than feature deliveries.

OpenAI's 28-day pledge defined a reset as a legitimate sprint event: a day on which, instead of shipping a feature, the team executes a complete usage quota reset for all users. The framing was generous: a reset is an act of goodwill, not a failure to ship.

The community's response to resets has been mixed. The first reset (Day 2) was met with surprise. By Day 15, community members were explicitly flagging that three resets in fifteen days represented a different kind of sprint than originally implied. Day 17's reset announcement prompted community discussion about whether the sprint's remaining eleven days would be able to deliver enough substantive improvements to meet the bar set by Days 11, 12, and 16.

The resets are not bad. But they are consuming sprint days that could be used for the efficiency improvements the community clearly values. A reset is worth, at most, a few days of extra quota. A 12-hour prompt caching improvement (Day 16) is worth every day of every session.

---

## The Unaddressed Token Spike

Separate from the scorecard is the outstanding token-usage anomaly reported by community member mulderpf: a 2.5 billion token consumption event in a single day under identical settings to sessions that previously consumed around 350 million tokens — a 7x multiplier.

As of Day 17, this report is over 48 hours old and has received no official OpenAI response.

The community has not missed the contrast. When v0.162.0 shipped with a VS Code extension regression (silent queue failure), OpenAI patched it within 24 hours in v0.162.1. That rapid response is now the reference point: billing anomalies with a 7x multiplier are receiving lower priority treatment than functional regressions.

Whether or not this comparison is fair — billing anomaly root-cause analysis is significantly harder than a patch to a JSON serialisation bug — the community's perception is forming in the absence of official communication. A 24-hour response saying "we are investigating" would cost nothing and would hold that perception in a much more stable place.

---

## What Days 18–28 Need to Deliver

Given the scorecard pattern, the community's stated priorities for the remaining eleven days are clear and consistent across multiple forum threads:

### Tier 1 — High impact, high approval expected
1. **Remote WSL control** — the most-upvoted open request in the Ideas forum, 18 upvotes as of 22 October. Explicitly framed as Codex catching up to Claude Code. If this ships in the final eleven days, it is a significant competitive signal.
2. **Quota increase for Pro/Team tiers** — consistently requested. Any substantive increase in hourly or daily rate limits will score high.
3. **Cost reduction for API users** — per-token pricing reduction or a new efficiency tier.
4. **Undo command** — 466 upvotes on the long-running request. Not just an undo for individual edits but an agentic undo — the ability to reverse the last agent action cleanly.

### Tier 2 — Medium impact, meaningful to subset of users
5. **Windows/WSL stability fixes** — several open issues, affects enterprise developers on Windows
6. **MCP tool discovery in the CLI** — browse and connect MCP servers without leaving the terminal
7. **v0.163.1 stable patch** — not a sprint feature, but shipping it would address the regression cluster and demonstrate that quality control is a priority

### Tier 3 — Lower priority (community data)
8. UI/UX polish, new themes, interface reorganisation — will score poorly against the backdrop of operational concerns

---

## The Deeper Question: Retention or Product?

Several community threads have named the sprint's subtext explicitly: the 28-day sprint is not a product exercise. It is a retention campaign.

The data behind this framing is the market share survey circulating in the community: Claude Code at 65.5% developer mind-share versus Codex at 29.5%. The sprint, in this reading, is OpenAI's visible response to that gap.

If the sprint is primarily about retention, the approval scores make complete sense. Developers considering switching away from Codex are doing so for operational reasons — cost, quota, reliability — not because they wish the UI were slightly different. The features that score 83-91% are precisely the features that make Codex easier to justify keeping.

The features that score 8% are the features that are visible in demos and announcements but do not change the developer's daily experience of the tool.

This reading is uncomfortable but coherent. It implies that OpenAI's sprint planning process is still optimising for demo visibility — features that are easy to screenshot and announce — at least in part, when the community is asking for infrastructure work that is unglamorous but decisive.

---

## Days 18–28: What to Watch

**The Day 18 announcement** will tell you more than any individual feature. If it is an efficiency or infrastructure improvement, the sprint's second half may close with a strong approval trajectory. If it is a UX feature or another reset, the community's watchful unease will deepen.

**Token spike resolution** — an official response, even if inconclusive, is overdue. Watch for this in the official Codex release notes and community forum.

**v0.163.1 stable** — the backport pace (two patches in nine hours on 22 October) suggests this is imminent. It will not be a sprint day, but it will affect the overall narrative of how OpenAI responds to regressions.

**Remote WSL control** — if this ships in the final eleven days, it changes the competitive framing significantly. If it does not ship, it will be the most-discussed absence in the sprint retrospective.

The sprint ends 1 November 2026. The community will be reading the final scorecard.

---


[^1]: ikaijua/Awesome-AITools sprint tracker: approval data for Days 1–16, October 2026.
[^2]: 28 Days of Shipping thread — community.openai.com/t/28-days-of-shipping-at-openai/1403897. Accessed 22 October 2026.
[^3]: Community sentiment weekly report, 22 October 2026, codex-resources/reddit-sentiment.
[^4]: Claude Code vs. Codex market share survey: 65.5% vs. 29.5%, community-sourced, October 2026.
[^5]: Undo command feature request: 466 upvotes, OpenAI Developer Community Ideas forum.
[^6]: Remote WSL control request (#14595): 18 upvotes, most-upvoted open request, OpenAI Developer Community Ideas forum.
[^7]: Token spike report (mulderpf): 2.5bn tokens vs. 350m baseline, October 2026. community.openai.com/c/codex/bugs.
[^8]: v0.162.1 patch turnaround reference: VS Code extension regression, 8–9 October 2026.
