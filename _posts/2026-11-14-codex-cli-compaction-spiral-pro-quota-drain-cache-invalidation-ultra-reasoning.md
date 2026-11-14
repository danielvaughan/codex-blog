---
title: "The Compaction Spiral: How Codex CLI Can Burn Your Entire Pro Quota Before Lunch"
date: 2026-11-14T08:00:00+00:00
last_modified_at: 2026-11-14T03:05:49Z
parent: "Articles"
nav_order: 1234
tags:
  - codex-cli
  - compaction
  - context-management
  - pro
  - quota
  - billing
  - cache
  - ultra-reasoning
  - bugs
  - performance
copy_edited: false
---

A fresh Codex CLI session. Two prompts in. Context already at 79%. By the time you notice something is wrong, your weekly Pro allowance — $200 worth — is gone.

This is not a hypothetical. It is the documented experience of multiple users across Issues [#60538](https://github.com/openai/codex/issues/60538), [#60791](https://github.com/openai/codex/issues/60791), and at least three prior duplicates (#60199, #59534, #59970). The root cause is a compaction feedback loop that is particularly severe with Ultra-tier reasoning models. Understanding it is the difference between a productive week and a support ticket.

## What Compaction Is Supposed to Do

Codex CLI's context compaction system exists to let long-running agentic sessions keep working even as conversation history approaches the model's context window limit. When context fills up, Codex compacts the earlier portion of the conversation — summarising or truncating it — so the session can continue without hard failure.

Under normal conditions, this is transparent and cheap. The compaction overhead is modest, and because the remaining conversation shares a cache prefix with subsequent requests, the token cache reuse rate stays high — typically 60% or above. You barely notice it happening.

The compaction spiral is what happens when those conditions break down.

## The Feedback Loop

The failure mode works like this:

1. A compaction event occurs — routine, nothing alarming.
2. Compaction **rewrites the beginning of the conversation history**.
3. The rewrite invalidates the cache prefix that all subsequent requests were anchored to.
4. The next request arrives and finds no valid cache hit. It must re-ingest the **entire** compacted conversation at full input-token cost.
5. The full re-ingestion pushes the context window back toward its limit.
6. Another compaction fires. Repeat.

Each iteration of the loop consumes tokens at input rates rather than cache rates. The difference is not marginal — cache reads typically cost a fraction of fresh input tokens. In a runaway loop with Ultra reasoning enabled, that multiplier compounds fast. Issue #60538 recorded 72 compaction events in a single session. Cache reuse had fallen to 15.7%.

## Ultra Reasoning Amplifies the Problem

Ultra-tier models such as gpt-6-astra in Ultra mode are inherently heavier on token consumption per exchange — that is part of what you pay for. Under normal circumstances this is acceptable: you get stronger reasoning at higher cost, and cache reuse keeps total spend predictable.

In a compaction spiral, Ultra reasoning removes all predictability. Each re-ingestion is not just expensive in absolute terms; it is expensive at a multiplier that scales with reasoning depth. The user in Issue #60538 lost their full weekly allowance in approximately eight hours of work. Issue #60791 documented a nine-hour Unity development task that consumed ~30% of the weekly quota while making zero measurable progress on the task itself — the tokens were burning on compaction overhead, not on the problem.

## Who Is at Risk

The spiral appears to affect sessions with these characteristics:

- **Long-running agentic tasks** — multi-hour sessions where context accumulates across many tool calls and model responses
- **Ultra reasoning tier** — the feedback is proportionally worse the more tokens a single exchange costs
- **Large working context** — large codebases, many file attachments, or verbose tool output logs that fill context quickly
- **macOS** — both confirmed reports are on macOS; it is unclear whether the underlying cause is platform-specific or whether Windows/Linux users are under-reporting

Standard conversational sessions — short prompts, quick responses, frequent new sessions — appear unaffected.

## Interim Mitigations

OpenAI has not published a fix or ETA as of 14 November 2026. There are several mitigations available now:

**Switch reasoning tier for long tasks.** Use Standard or Extended reasoning rather than Ultra for multi-hour agentic sessions. Reserve Ultra for discrete, bounded reasoning problems — architecture analysis, complex debugging — where you can bound the session length and restart cleanly.

**Break sessions earlier.** The spiral's severity scales with session length and total context volume. For long agentic tasks, divide work into discrete bounded sessions rather than one continuous run. Restart Codex between major phases.

**Monitor cache reuse.** Normal cache reuse is 60%+. If you see it drop below 30% in the token usage display, treat it as a warning sign. Exit the session and restart before the loop accelerates.

**Watch context percentage on first prompts.** A healthy new session should not hit high context percentages within the first two prompts. If the context indicator climbs steeply from the start, something is already wrong with the cache state. Close and reopen.

**Avoid the Ultra reasoning + large codebase combination.** The highest-risk configuration is Ultra reasoning on a codebase with significant ambient context. If the task genuinely requires Ultra, reduce the ambient context — close unnecessary files, prune tool call logs, use focused subagents for bounded subtasks.

## The Transparency Problem

Underlying the technical failure is a transparency failure. Codex's billing model is consumption-based, but the consumption feedback during a session is limited: a context percentage bar and a quota percentage consumed, neither of which gives you enough warning before a runaway compaction cycle has already cost you significantly.

The community thread titled *"Codex froze and burned nearly 30% of my weekly limit"* captures the user experience well: not a dramatic failure, but a silent drain. No error. No warning. The session appeared to be working. The quota disappeared.

This is happening in the same week that OpenAI cut the Pro plan's computational multiplier from 20x to 10x with no documented verification mechanism — users cannot independently confirm what multiplier is being applied to any given request. The combination of an opaque multiplier cut, an undocumented background Luna reviewer (see Issue #59835), and a compaction loop that can silently drain weekly quota in hours is producing real trust erosion. Multiple threads this week cite these failures as reasons to evaluate Claude Code's billing model, which publishes concrete token limits per tier.

## What Should Change

The technical fix is well understood: compaction must not fully invalidate the cache prefix. A correct implementation would compact only history segments that fall outside the shared prefix window, or would rebuild the prefix incrementally after compaction rather than triggering a full re-ingestion. The engineering challenge is non-trivial, but the failure mode is deterministic and reproducible. Issues #60538 and #60791 provide sufficient data for a root cause fix.

The transparency fix is simpler: surface a compaction event count and cache reuse rate in the session status display. Users should be able to see that something is burning tokens at ten times the expected rate before the damage is done.

Until then: break your sessions early, stay off Ultra for long agentic work, and watch the cache numbers.

---

*Sources: GitHub Issues #60538, #60791, #60199, #59534, #59970; community thread "Codex froze and burned nearly 30% of my weekly limit" (Nov 13, 169 views). Version current as of Codex CLI v0.168.0 (stable, Nov 13 2026).*
