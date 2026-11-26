---
title: "Paperclip Budget Controls: Why a Hard Stop Enforced on Recorded Spend Is Not a Spending Cap"
date: 2026-11-25T08:00:00+00:00
last_modified_at: 2026-11-26T15:07:39Z
tags:
  - codex-cli
  - paperclip
  - multi-agent
  - budget-controls
  - cost-governance
  - orchestration
  - enterprise
rating: 3.5
depth: 3
originality: 4
accuracy: 4
substack: 3
copy_edited: true
---

Paperclip, the open-source orchestrator that can run Codex as one of its agents, has "hard stops" in its budget policy. Its own README then says that enforcement "uses recorded spend" and that "usage reporting and in-flight work can delay a stop". Those two statements together mean a hard stop is a control on what has been recorded, and not a ceiling on what the provider will bill.

This article reads what Paperclip documents about budgets, separates that from what the community coverage adds, and turns it into practical settings for anyone running Codex under an orchestrator. My earlier backlog note described the mechanism as heartbeat-based enforcement. The sources do not say that, and the corrected picture is below.

---

## What Paperclip Documents

All quotations are from the [Paperclip README](https://github.com/paperclipai/paperclip) as read on 25 November 2026.

- **Scopes.** Features list "Company, agent, and project budgets." The org-chart section adds that agents have "roles, titles, reporting lines, permissions, and budgets."
- **Tracking.** "Token and cost tracking by company, agent, project, goal, issue, provider, and model."
- **Policy.** "Scoped budget policies with warning thresholds and hard stops." Governance lists "budget hard-stops" next to "agent pause/resume/terminate".
- **The stated limitation.** "Enforcement uses recorded spend; usage reporting and in-flight work can delay a stop."
- **Stated purpose.** Without it, "runaway loops waste hundreds of dollars of tokens and max your quota before you even know what happened."

Codex is listed as a supported runtime. The README gives a test-drive command of the form `npx paperclipai test-drive --harness codex`, and notes that Codex subscription sign-in needs the Codex CLI on the Paperclip host.

---

## How the Check Works: Admission and Settlement

The repository's [heartbeat internals document](https://github.com/paperclipai/paperclip/blob/master/doc/heartbeat-internals.md) describes the accounting design more precisely than the README. Three points matter:

1. "Budget admission and settlement share the company accounting lock." A run is checked before it starts (admission) and its charge is recorded afterwards (settlement).
2. "Running usage is an internal exposure projection, not a provisional ledger charge." While a run is in progress, Paperclip estimates exposure. It does not book a charge until settlement.
3. "Unknown provider outcomes retain that capacity until accounting establishes the charge," and "the existing unpriced-usage policy controls admission until their terminal charge is known."

So the mechanism is admission and settlement against a ledger, and heartbeats are the wake-up queue that carries a budget check. The README's description of heartbeat execution lists "budget checks" as one step in a "DB-backed wakeup queue", followed by workspace resolution, secret injection and adapter invocation. The check sits at admission. A run that has already started is bounded by the stop machinery, which the same document describes as covering "Stop, agent and budget cancellation".

The document also says what a stop does not do: a confirmed operator Stop "does not automatically restart the task", and budget limits "keep their own gates" alongside recovery holds and approvals.

---

## Why a Stop Lags

The README does not enumerate the causes. From the documented design, three follow, and I mark them as inferences:

- **Settlement trails spend.** The ledger can only stop what it has seen. Tokens consumed by a long in-flight run are projected exposure until the run settles.
- **Reporting depends on the adapter.** The README says "usage reporting" can delay a stop. An agent runtime that reports usage late, or cannot price a model, leaves the ledger behind the provider.
- **A stop is a request to cancel.** Cancellation needs the adapter to acknowledge it. The benchmark notes in the heartbeat document say "stop acknowledgement" must be measured separately from database and network latency, which implies it is a distinct delay.

---

## What the Community Review Adds

AICodeKing's [video review of Paperclip](https://www.youtube.com/watch?v=RknGPBeA6AQ) (6 October 2026; transcript in `resources/transcripts/Paperclip_Agent_Team_AICodeKing.md`) repeats the README's limitation and draws the practical conclusion: "I wouldn't treat a budget setting as a guarantee that the provider bill will land exactly on that number." His own approach is to "run one small task, look at the usage, and then decide how much room to give each agent."

He also notes that a reviewer going back and forth with an implementer "adds usage, even if the app itself is fairly simple", and that timer heartbeats are off by default for new agents, which he calls "a sensible choice" because an agent that wakes every few minutes to find nothing to do creates "activity without getting much out of it". That is his reading of a default, and the README confirms the underlying behaviour: agents "wake for assigned work and follow-up messages", with timer heartbeats optional.

The video is a single reviewer's account, not a measurement. I found no published figures for how far a stop overshoots, so none are given here.

---

## Practical Settings for Codex Under an Orchestrator

These are recommendations, not Paperclip features.

```mermaid
flowchart LR
    A[Provider limit<br/>the real ceiling] --> B[Paperclip hard stop<br/>set below it]
    B --> C[Warning threshold<br/>set below the stop]
    C --> D[Per-project budgets<br/>for experiments]
```

1. **Leave headroom under the real limit.** Treat the hard stop as a trip-wire with slack, and set it below the amount you cannot exceed. How much slack depends on how large a single run can be, which you can learn by observing one.
2. **Measure first.** Follow the reviewer's advice: run one small task, compare Paperclip's recorded spend with the provider's usage view, and size budgets from the gap.
3. **Use narrow scopes for experiments.** Project and agent budgets confine a runaway to one scope before the company budget is touched.
4. **Put a ceiling outside the orchestrator.** Codex has its own controls. Where your plan or workspace offers spend limits, set them at the provider. They are enforced by the party that bills you. Compare with the in-Codex tools covered in the [rate-limit and budget articles in this library](https://danielvaughan.com/codex-resources/articles/).
5. **Know the manual stop.** Paperclip's pause and terminate controls are the fastest response to a loop you can see. Learn where they are before you need them.
6. **Keep timer heartbeats off** unless an agent has a reason to poll.

---

## What I Could Not Verify

- The Paperclip documentation site (`docs.paperclip.ing`) and `paperclip.ing` were not reachable from the environment that wrote this article, so the budgets guide and the changelog entry cited in the backlog were not read. Everything above comes from the GitHub repository and the video transcript.
- I did not run Paperclip, so the behaviour of a stop against a live Codex session is documented design, not observation.
- The default warning percentage is not stated in the sources read.

---

## Citations

1. Paperclip README, github.com/paperclipai/paperclip (read 25 November 2026).
2. Paperclip `doc/heartbeat-internals.md`, github.com/paperclipai/paperclip/blob/master/doc/heartbeat-internals.md (read 25 November 2026).
3. AICodeKing, "Paperclip: Run a Small AI Agent Team (Claude Code + Codex) Without the Chaos", youtube.com/watch?v=RknGPBeA6AQ, 6 October 2026.
