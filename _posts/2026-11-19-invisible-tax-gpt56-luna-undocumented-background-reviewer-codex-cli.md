---
title: "The Invisible Tax: GPT-5.6 Luna and Codex CLI's Undocumented Background Reviewer"
date: 2026-11-19T14:00:00+00:00
last_modified_at: 2026-11-20T13:07:16Z
tags:
  - codex-cli
  - pricing
  - gpt-6-luna
  - quota
  - transparency
  - guardian
  - billing
  - pro-plan
rating: 4.5
depth: 5
originality: 4
accuracy: 5
substack: 4
copy_edited: true
---

Something odd started appearing in Pro subscriber usage stats in mid-November 2026. Users opened their Codex dashboard and found GPT-5.6 Luna listed — a model they had never chosen, never configured, and never deliberately invoked. A community thread drew 696 views and four replies before the pattern became clear: Codex CLI had been running a background process called `codex-auto-review` without telling anyone.[^1]

This article explains what that process is, what it costs, why it matters now, and what you can do about it.

---

## What Is `codex-auto-review`?

`codex-auto-review` is a background process introduced quietly in a recent Codex CLI release. It runs automatically at two points in a session lifecycle:

1. **Guardian security review** — after every agentic task completes, before the result is surfaced to the user, a lightweight review pass checks whether the agent's actions crossed any configured security or scope boundaries.
2. **Workspace reconciliation** — when a session reconnects after interruption (network drop, cloud session restart, idle reconnect), the process reconciles current workspace state against the session's internal model of what it had done.

Both passes run against GPT-5.6 Luna. Luna is fast and cheap relative to GPT-6 Astra or Sol — which is precisely why it was chosen for background review work. The problem is that this choice was never documented in onboarding, in the settings panel, or in any release note visible to subscribers. Users found out through their billing dashboard.

---

## The Token Cost

Luna's pricing is \$0.20 per million input tokens and \$1.20 per million output tokens — genuinely inexpensive at that rate. For a single session, the background review cost is small.

The issue is scale and compounding. OpenCode.ai estimated that `codex-auto-review` was consuming between 6 billion and 20 billion Luna tokens per day across the Codex user base as of November 2026.[^2] At the individual level, a heavy Pro subscriber running long agentic sessions with frequent workspace reconnects can accumulate tens of millions of background Luna tokens in a week without any visible indicator in the TUI.

The dollar cost alone is not the main concern for most Pro subscribers — the flat-rate plan absorbs it. The real concern is **multiplier-weighted quota consumption**.

---

## The Multiplier Problem

Codex Pro plans use a computational multiplier to convert raw usage into quota: a session on GPT-6 Astra Ultra uses more quota per token than a session on GPT-6 Luna. The exact multiplier per model is not published, but the community has established through experimentation that background Luna usage is counted against the weekly quota using a non-zero multiplier — it is not exempt from the usage accounting system.

This came to a head in the same week the Pro 200 multiplier was cut from 20× to 10×. Subscribers were already absorbing a 50 per cent reduction in effective headroom when they discovered that an invisible background process had been silently competing for that reduced headroom throughout the previous weeks.[^3]

The dashboard displays percentage of weekly limit consumed, not underlying per-model accounting. Subscribers have no way to isolate how much of their consumed percentage came from `codex-auto-review` versus their primary model work. That opacity — not the token cost itself — is what generated the community frustration.

---

## Three Compounding Problems at Once

November 2026 landed three separate quota-drain mechanisms simultaneously on Pro subscribers:

1. **The compaction loop** — cache prefix invalidation during compaction events drives input token re-ingestion costs into a positive feedback loop, draining quota faster than expected on long sessions with Ultra reasoning.[^4]
2. **The multiplier cut** — Pro 200 effective headroom halved from 20× to 10× with no independent verification mechanism.
3. **The background reviewer** — `codex-auto-review` consuming quota silently across all sessions, not just the complex ones.

No single one of these would dominate. The convergence of all three in the same billing window produced the reports of Pro subscribers exhausting their weekly allowance in a fraction of the expected time. When you cannot isolate the cause — because the dashboard does not expose per-process or per-model accounting — the rational response is to assume the worst about all three.

---

## What OpenAI Has and Has Not Said

As of 19 November 2026, there has been no official acknowledgement of `codex-auto-review` as a user-facing feature requiring documentation. GitHub Issue #59835, which identified the background process and proposed switching it to GPT-6 Luna (a newer, cheaper model) rather than GPT-5.6 Luna, remains open with no response from the Codex team.[^5]

The community proposal in that issue is reasonable: if a background review process must run, it should:

- Be disclosed during onboarding and surfaced in the model/session information pane
- Use the cheapest capable model available (GPT-6 Luna at \$0.10/\$0.50 per million tokens rather than GPT-5.6 Luna at \$0.20/\$1.20)
- Be isolatable in usage accounting so subscribers can see its contribution to quota consumption
- Be configurable — with an opt-out for users who have implemented their own post-task review hooks in AGENTS.md

---

## What You Can Do Now

**Audit your usage breakdown.** Check your usage page at platform.openai.com for model-level breakdown. If GPT-5.6 Luna appears with significant usage on a week when you did not explicitly select it, `codex-auto-review` is running.

**Review your Guardian configuration.** If you have Guardian enabled in your Codex CLI config (the default for Pro accounts), `codex-auto-review` runs after every task. If you are running lightweight, trusted local tasks where Guardian review adds no value, you can configure a narrower Guardian scope to reduce review frequency:

```toml
[guardian]
enabled = true
review_on = ["file_delete", "network_call", "process_spawn"]
# omit "task_complete" to suppress end-of-task auto-review
```

**Add explicit review to AGENTS.md instead.** For users who want post-task review but want it on their own terms, add an explicit review step to your AGENTS.md. This gives you control over which model runs the review, what scope it covers, and when it runs — and makes the cost visible in your primary model's usage, not hidden in a background process.

```markdown
## Post-task review
After completing any task that modifies files outside the current working directory,
run a brief scope check: list changed files, confirm they are within the agreed scope,
flag any unexpected additions.
```

**Monitor for the fix.** The OpenCode.ai proposal to migrate `codex-auto-review` from GPT-5.6 Luna to GPT-6 Luna would roughly halve the per-token cost of the background process. Watch the v0.172.0 sprint release notes for any mention of auto-review changes.

---

## The Trust Dimension

The technical cost of `codex-auto-review` is small. The transparency cost is not.

Codex CLI is a Pro product at \$200 per month. At that price point, subscribers have a reasonable expectation that every process consuming their quota is documented. The comparison point in community threads is not theoretical: Claude Pro publishes concrete session limits per tier. Anthropic's transparency on billing mechanics has become a repeated competitive differentiator in precisely the threads where Codex's quota behaviour is being questioned.[^6]

The fix is not complicated. Disclose the background process. Surface it in the UI. Make it configurable. That would not eliminate the quota cost, but it would eliminate the trust cost — which, in a week that also included a multiplier cut and a compaction regression, is the more urgent problem to solve.

---

*Research date: 2026-11-19 | Sources: community.openai.com, GitHub codex repository, OpenCode.ai token analytics*

[^1]: Community thread — "Luna appearing in my usage stats even though I never selected it" (696 views, 4 replies, community.openai.com/c/codex/37, Nov 13 2026)
[^2]: OpenCode.ai token analytics, cited in community thread discussion, Nov 13 2026. Figures represent estimated aggregate across active Codex Pro/Business/Enterprise users.
[^3]: Community thread — "Pro 200 cut from 20x to 10x: how can customers verify what these multipliers actually mean?" (625 views, 7 replies, Nov 13 2026)
[^4]: GitHub Issue #60538 — compaction spiral, 72 events, 15.7% cache reuse (macOS 14.7.8, Nov 2026). See also: Issue #60791, #60199, #59534, #59970.
[^5]: GitHub Issue #59835 — GPT-5.6 Luna as undocumented background auto-reviewer; proposal to migrate to GPT-6 Luna and add user-facing disclosure.
[^6]: Community megathread — usage limits and Claude Code migration (1,048+ replies, Nov 2026). Luna transparency gap cited explicitly in replies 847, 902, 1,003.
