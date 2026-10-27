---
title: "The Streaming Interrupt Paradox: How v0.163.0's Headline Feature Breaks Itself"
date: 2026-10-22T07:00:00+00:00
last_modified_at: 2026-10-27T10:05:16Z
parent: "Articles"
nav_order: 1194
tags:
  - codex-cli
  - v0.163.0
  - bugs
  - regression
  - mid-response-steering
  - tui
  - reliability
copy_edited: true
---

# The Streaming Interrupt Paradox: How v0.163.0's Headline Feature Breaks Itself

![Sketchnote diagram for: The Streaming Interrupt Paradox: How v0.163.0's Headline Feature Breaks Itself](/sketchnotes/articles/2026-10-22-streaming-interrupt-paradox-mid-response-steering-regression-codex-v0163.png)

**Affected version:** v0.163.0 stable (15 October 2026)
**Bug status:** Open — community-reported, one week active, 7+ replies
**Companion issues:** TUI `-32603` bootstrap failure (macOS); all-models-at-capacity regression
**Expected fix track:** v0.163.1 stable or v0.164.0 stable

---

When v0.163.0 shipped on 15 October 2026, its headline change was unmistakable in the release notes: mid-response steering, the ability to send a follow-up prompt while Codex is actively generating a response, was switched from opt-in to on by default.

One week later, the community has surfaced a regression that makes that headline difficult to celebrate: sending a follow-up prompt mid-stream does not steer the response — it terminates it. The feature and the bug are the same user action.

This is the streaming interrupt paradox.

---

## What Mid-Response Steering Is Supposed to Do

Mid-response steering was introduced in the v0.163.0 alpha sprint as a response to one of the most common frustrations with agentic coding sessions: the inability to intervene once a task has started.

In a typical Codex session without mid-response steering, a user submits a prompt, Codex begins executing, and the user watches. If the execution starts going in the wrong direction — wrong file being modified, wrong approach being taken, wrong interpretation of the requirement — the user's only option is to wait for completion and then correct, or to interrupt the entire session and start over.

Mid-response steering changes this. In principle, the user can observe Codex mid-execution, type a follow-up that says "stop — you're editing the wrong file" or "use the other approach instead", and Codex incorporates that guidance without the current turn completing. The execution steers rather than stops.

This is a meaningful capability shift. It moves Codex closer to a collaborative model where the human and the agent are working together in real time, rather than the human submitting and waiting.

---

## The Regression

In practice, as reported by community members in the week following v0.163.0 stable, sending a follow-up prompt during an active stream does not steer the response — it halts it.

The observed behaviour:

1. User sends an initial prompt. Codex begins generating a response or executing a task.
2. User types a follow-up while the stream is active.
3. The active stream stops. The current turn's output is truncated at the point where the follow-up was submitted.
4. Codex either starts a fresh response to the follow-up (treating it as a new turn, not a mid-stream steer) or displays an error state.

The steering mechanism is not functioning. Instead of receiving the follow-up as guidance to incorporate into the current execution, Codex is treating it as a turn boundary — terminating the current generation and starting fresh.

---

## Why This Is Particularly Damaging

Most regressions in a release cycle represent a capability that worked before and stopped working. The streaming interrupt regression is different: it represents the failure of a capability that was explicitly presented as working for the first time.

Users who install v0.163.0 specifically to try mid-response steering will encounter the regression immediately. Their first experience with the headline feature is a failure mode. And because the failure is silent — Codex appears to respond normally to the follow-up, just as a new turn — users may not immediately recognise that the steering did not work as intended. They may simply notice that their follow-up corrections do not take effect as expected.

This is the worst kind of regression: one that does not obviously signal its own failure, and which is discoverable only by users who already understand how the feature is supposed to behave.

---

## The Broader Regression Cluster

The streaming interrupt is not isolated. In the week following v0.163.0 stable, the community has identified three simultaneous regressions that are each blocking daily use:

### TUI `-32603` Bootstrap Failure (macOS)

Affected users see a `-32603` JSON-RPC error when Codex initialises on macOS. The failure occurs at startup, before the user is able to enter any prompt. On affected systems, Codex is completely unusable.

This is an onboarding blocker: a user who installs v0.163.0 fresh on macOS may encounter the failure before they have run a single successful session. It is also a trust barrier — a startup crash is the most visible and alarming category of failure.

The v0.163.0 backport track is actively targeting this issue. `v0.163.0-alpha.11.2` (shipped 22 October 2026, 06:41 UTC) includes three TUI startup commits from Benjamin Carlsson and Eric Traut that appear directed at this failure:

- `Prepare TUI startup to accept multiple backend sessions` (PR #56097)
- `Keep TUI preferences editable when execution settings are invalid` (PR #56091)
- `Honor server approval reviewer defaults on fresh TUI startup` (PR #56090)

A v0.163.1 stable patch appears likely within days.

### All-Models-At-Capacity on Plus Account

A separate regression reported on 22 October: users on a single Plus account are encountering persistent "all models at capacity" errors, even for basic tasks. This is a quota enforcement or routing issue at the service level rather than a local CLI problem, but it manifests as a daily-usage blocker for affected users.

### Token Usage Spike (7x Baseline)

Community member mulderpf reported a 2.5 billion token usage event under identical settings to previous sessions that produced around 350 million tokens — a 7x multiplier. This report is over 48 hours old as of this writing and has received no official OpenAI response.

The spike's cause is unconfirmed. Theories in the community include a context compaction regression, a prompt repetition loop in the new mid-response steering code path, or a monitoring artefact. Without an official response, the uncertainty is worse than the number itself: users cannot know whether their own usage is affected.

---

## The Pattern

These three regressions arriving together tell a coherent story about v0.163.0's release cycle.

v0.163.0 was an ambitious release. Its alpha sprint ran to at least eleven iterations, significantly longer than typical. The headline features — mid-response steering default-on, `/config`, Daybreak, parallel tools — were all meaningful product changes, not incremental fixes. The sprint was shipping faster than its quality controls could fully validate.

The regressions are consistent with that picture. They are not in obscure edge-case paths. They are in the most visible parts of the product:
- Startup (the first thing a user experiences)
- The headline feature (the first thing a new user tries)
- Token billing (the first thing an API user checks)

That three high-visibility regressions arrived simultaneously suggests that v0.163.0's alpha sprint prioritised feature completion over regression testing.

---

## Daybreak Instability as Context

One additional v0.163.0 pain point that community members have flagged: Daybreak — the Codex TUI's visual theme — has had its default state changed in three consecutive releases.

- v0.161.0: Daybreak opt-in
- v0.162.0: Daybreak opt-out
- v0.163.0: Daybreak default-on

Three releases, three different defaults. This is not a regression in the functional sense, but it is a symptom of the same underlying issue: a sprint-driven release cadence that is adjusting user-visible settings faster than users can stabilise their configurations.

---

## What To Do Right Now

**If you are on v0.163.0 on macOS and seeing `-32603` on startup:**
Downgrade to v0.162.1 until v0.163.1 ships:
```bash
npm install -g @openai/codex@0.162.1
```

**If you are relying on mid-response steering:**
The feature is not working as intended in v0.163.0. Treat it as unavailable for the time being. Monitor the release notes for a v0.163.1 fix.

**If you are watching your API token consumption:**
Check your usage dashboard now. If you see anything approaching a 7x spike versus your baseline, pause API-heavy Codex tasks until the issue is officially addressed.

**If you are evaluating Codex vs. alternatives for Q4 tool commitment:**
The regression cluster is real but the backport pace (two v0.163.0 patches in under 9 hours on 22 October) suggests active remediation. A v0.163.1 stable release is the data point worth waiting for before drawing conclusions.

---

## What to Watch For

- **v0.163.1 stable** — expected within days. The backport patch rate (alpha.11.1 and alpha.11.2 within 9 hours) is consistent with a pre-stable patch cycle.
- **Official response to the token spike** — every additional day of silence makes this a larger trust issue.
- **Streaming interrupt fix** — the most architecturally significant of the three regressions. A fix here will confirm that mid-response steering works as described in the v0.163.0 release notes.

---


[^1]: v0.163.0 stable release, 15 October 2026. Mid-response steering default-on confirmed in release notes.
[^2]: Streaming interrupt regression: community reports, week of 15–22 October 2026. Status: open, 7+ replies.
[^3]: TUI `-32603` bootstrap failure: community reports, October 2026. Onboarding blocker on macOS.
[^4]: v0.163.0-alpha.11.2, 22 October 2026, 06:41 UTC. TUI startup fixes: PRs #56097 (Benjamin Carlsson), #56091, #56090 (Eric Traut).
[^5]: Token usage spike (mulderpf): 2.5bn tokens vs. 350m baseline, reported October 2026. Community.openai.com/c/codex/bugs.
[^6]: All-models-at-capacity regression: community reports, 22 October 2026.
[^7]: Daybreak default state history: v0.161.0 opt-in, v0.162.0 opt-out, v0.163.0 default-on.
