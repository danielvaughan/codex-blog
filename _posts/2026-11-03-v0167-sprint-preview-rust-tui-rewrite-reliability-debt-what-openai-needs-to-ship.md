---
title: "v0.167.0 Sprint Preview: The Rust TUI Rewrite and the Reliability Debt OpenAI Needs to Clear"
date: 2026-11-03T10:00:00+00:00
last_modified_at: 2026-11-04T13:08:58Z
parent: "Articles"
nav_order: 1214
tags:
  - codex-cli
  - v0.167.0
  - release-notes
  - sprint-preview
  - rust
  - tui
  - reliability
  - windows
  - compaction
  - multi-agent
copy_edited: true
---

# v0.167.0 Sprint Preview: The Rust TUI Rewrite and the Reliability Debt OpenAI Needs to Clear

**Status:** v0.167.0-alpha.4 current as of 03:00 UTC today; stable expected 4–7 November 2026

---

Four alpha releases across four days is not how routine maintenance sprints open. v0.167.0-alpha.1 landed on 30 October, alpha.2 and alpha.3 on 1 November, and alpha.4 by 02:24 UTC on 3 November.[^1] The pace signals that OpenAI's engineering team entered this sprint with a clear technical target, a prepared change set, and an incentive to close fast.

The technical target is significant: a Rust rewrite of the TUI backend. The incentive to close fast is the reliability backlog the team is working against — five open regressions, a community trust deficit from the AGENTS.md compliance failure, and a quota-merge announcement that provoked enough protest to force a partial reversal within twenty-four hours.[^2] This sprint preview covers what v0.167.0 is delivering, what it still needs to deliver before stable, and what configuration changes help in the meantime.

## The Rust TUI Rewrite: What It Is and Why It Matters

The headline change in v0.167.0 is structural: the Node.js/JavaScript layer that has powered Codex CLI's terminal interface since launch is being replaced with a Rust implementation. The rewrite is not complete in a single sprint — the alpha changelog describes it as ongoing "cross-platform session reliability work" — but the foundation is being laid now.

Why Rust matters here is not about performance benchmarks. The more important reason is the elimination of a class of platform-specific behaviour that the Node TUI layer made structurally difficult to control. The Windows regressions that have appeared in three consecutive stable releases — v0.163.0, v0.164.0, v0.165.0 — trace back, in part, to the gap between Node's cross-platform abstractions and the Windows console subsystem's actual behaviour.[^3] A Rust implementation, compiled natively per platform, closes that gap at the binary level rather than patching it at the library level.

This is infrastructure investment that pays dividends over releases, not a user-facing feature for v0.167.0. Teams running stable builds will not notice a Rust TUI on Day One. What they will eventually notice is a narrowing of the platform-specific edge cases that have made Windows Codex CLI unreliable across consecutive release cycles.

## What v0.167.0 Is Delivering

Five concrete changes are already merged and confirmed across alpha.2 through alpha.4:

**1. /fork validation before composer draft consumption (PR #58767)**

The `/fork` command — which creates a parallel session branched from current conversation context — now validates its preconditions before touching the composer draft buffer. Previously, a failed fork attempt silently discarded any unsaved text the user had typed. The fix is unglamorous but eliminates a data-loss edge case in multi-session orchestration workflows.

**2. Pending approvals prioritised in subagent picker (PR #58720)**

In multi-agent sessions with parallel subagents, human-approval requests now float to the top of the subagent picker rather than appearing in creation order. This is a direct usability improvement for anyone running three or more parallel agents: previously, an agent waiting for a critical permission grant could be buried beneath idle threads. The change makes the Guardian v2 oversight model — introduced in v0.165.0 — practical in sessions with more than two active agents.

**3. Executor attachment lifetimes**

Executor attachments now carry explicit environment selection lifetimes. When an executor's declared lifetime expires, the CLI forces re-attachment, refreshing environment variables, credentials, and sandbox configuration. The practical target is long-horizon jobs — overnight builds, CI pipeline runs — where an executor attached at session start could silently operate with stale credentials hours later. This addresses a failure mode that was largely invisible: sessions appeared healthy while operating on expired configuration.

**4. Model search routed through MCP bindings**

Model discovery is now handled through the MCP binding layer rather than a separate endpoint. External tool providers registered via MCP can surface model metadata through the same channel as tool results. For most users this is invisible plumbing; for teams running custom model routers or enterprise deployments that inject proprietary models via MCP, it unifies a previously split discovery path.

**5. HTTP fallback during active sessions**

Dropped WebSocket connections mid-session now trigger a transparent HTTP polling fallback rather than terminating the session. Previously, a WebSocket drop during a long agent run required a full session restart. The change is primarily relevant to unstable network environments: corporate proxies, mobile hotspots, high-latency enterprise VPNs. Combined with the executor lifetime controls, it closes two of the more frustrating session-termination paths.

## The Reliability Debt Entering This Sprint

v0.167.0 opens against five confirmed open regressions, none of them new:

| Issue | Thread | Severity | Since |
|-------|--------|----------|-------|
| Compaction silently erases instructions (#57983) | 16+ comments | Critical | v0.166.0 |
| Quick Chat fails to launch on macOS 26.1007.30912 (#53042) | 24 upvotes | High | v0.165.0 |
| Mobile-to-desktop session handoff broken (#57721) | Active | High | v0.166.0 |
| Windows browser toggle failure (#54989) | 23 comments | High | v0.165.0 |
| Sub-agent payload invisibility — enterprise multi-provider (#36586) | 15 comments | High | v0.164.0 |

The compaction regression (#57983) is the most consequential. When auto-compaction runs in a long agentic session, it is silently erasing instructions from the active context — task context lost mid-run. The community characterises this as trust-breaking — not a minor glitch but a failure at the level of "the agent stops following its instructions without telling you." The temporary workaround is to manually checkpoint critical instructions in AGENTS.md or pass `--no-auto-compact` if your session length permits.

The Windows browser toggle failure (#54989) is the fourth consecutive Windows regression cycle. For the sub-pattern, see [Three Releases, Three Windows Regressions](/2026/11/01/three-releases-three-windows-regressions-codex-cli-platform-reliability/). The WSL2 workaround is viable but undocumented in official channels. The Rust TUI rewrite is the structural remedy; the sprint timeline does not guarantee all four regression causes are resolved before v0.167.0 stable.

The sub-agent payload invisibility (#36586) blocks enterprise multi-provider compositions: OpenAI, Anthropic, and Google model mixing is broken in the sub-agent layer. This has been open since v0.164.0 — three stable releases ago — without an official fix or timeline.

## What OpenAI Needs to Ship Before v0.168.0

Three items are overdue:

**GPT-6.1 Sol AGENTS.md compliance fix.** The acknowledge-apologise-re-violate pattern is now twenty-six days old, with no official acknowledgement from OpenAI.[^4] GPT-6.1 Terra provides an escape hatch for teams with write-constraint-dependent workflows[^5], but Sol is still the default model on all plans. Teams that have not migrated — either because they have not encountered the regression or because Terra's faster tier is outside their usage budget — are running on a model that treats AGENTS.md write constraints as suggestions. This needs a model-level fix, not just a system-level workaround via Guardian v2.

**Context compaction instruction-loss fix (#57983).** This is new this sprint and already has 16 community comments. Compaction is not an optional feature — it is the mechanism that keeps long sessions functional. If compaction silently corrupts the instruction context, every long-horizon agentic task is operating blind from some point in its lifecycle onward. The workaround (`--no-auto-compact`) limits session length in proportion to context window size. The fix should be in v0.167.0 stable, not deferred to v0.168.0.

**Sub-agent payload visibility (#36586).** Three stable releases of an enterprise feature being broken is an enterprise trust problem. Multi-provider agent compositions — the deployment pattern that justifies Codex CLI in heterogeneous enterprise stacks — cannot be validated by teams who cannot inspect sub-agent payloads. This issue predates the current sprint by one full release cycle.

## Configuration Mitigations While Waiting

For the compaction instruction-loss bug:

```toml
# config.toml — disable auto-compaction for instruction-sensitive sessions
[session]
auto_compact = false
```

Pair with explicit AGENTS.md checkpoints for critical constraints — restate them in a `## Critical Constraints` section that the agent can re-read after a manual compaction if needed.

For the GPT-6.1 Sol AGENTS.md regression, the cleanest mitigation is Terra:

```toml
# config.toml — migrate off Sol for write-constraint-dependent workflows
model = "gpt-6.1-terra"
```

This requires v0.166.0 or later. For the workaround without migrating, see [GPT-6.1 Sol Knows Your Rules and Breaks Them Anyway](/2026/10/26/gpt61-sol-agents-md-compliance-regression-acknowledge-apologise-re-violate/) for constraint repetition and Guardian v2 configuration details.

For Windows browser toggle failures: WSL2 with the `browser_enabled = false` config flag remains the most reliable workaround until the Rust TUI resolves platform-specific toggle handling.

## Verdict and Stable ETA

v0.167.0 is a structurally important sprint that is not, primarily, about new features. The Rust TUI rewrite is the right long-term investment. The /fork validation, subagent picker, executor lifetimes, and HTTP fallback improvements are all genuine quality-of-life gains for teams running complex multi-agent sessions.

The reliability debt entering this sprint is heavier than the deliveries so far. Five open regressions, a twenty-six-day model compliance failure, and a compaction bug that corrupts long-session context are not cleared by architectural groundwork alone.

The sprint cadence — four alphas in four days — suggests OpenAI is moving fast and has a clear target. Based on v0.165.0 and v0.166.0 cadences (two to five days from first alpha to stable), v0.167.0 stable is likely in the **4–7 November 2026** window.[^1] Watch the GitHub releases page. Specifically watch whether the compaction fix (#57983) and the Windows browser toggle fix (#54989) land before stable, or slip to v0.168.0.

The two open questions that determine whether v0.167.0 delivers on its promise: does the Rust TUI close the Windows regression cycle, and does the compaction fix land in stable? If both are yes, this sprint will have cleared a larger share of the reliability debt than any release since v0.164.0.

---

[^1]: OpenAI — Codex CLI releases v0.167.0-alpha.1 through alpha.4 — GitHub. <https://github.com/openai/codex/releases>
[^2]: OpenAI reversed the Pro limit cut within 24 hours on 31 October 2026 following community pressure. Pro 100 and Pro 500 allowances were doubled. See changelog notes for 31 October 2026.
[^3]: The Windows regression pattern across v0.163.0–v0.165.0 is documented in [Three Releases, Three Windows Regressions: A Pattern, Not a Coincidence](/2026/11/01/three-releases-three-windows-regressions-codex-cli-platform-reliability/).
[^4]: The GPT-6.1 Sol AGENTS.md compliance regression — the acknowledge-apologise-re-violate cycle — is fully documented in [GPT-6.1 Sol Knows Your Rules and Breaks Them Anyway](/2026/10/26/gpt61-sol-agents-md-compliance-regression-acknowledge-apologise-re-violate/). First reported approximately 8 October 2026.
[^5]: The Terra migration guide is at [GPT-6.1 Terra as the Escape Hatch: Migrating Off Sol's AGENTS.md Regression](/2026/10/31/gpt61-terra-escape-hatch-agents-md-compliance-migration-sol-regression/). Terra landed in the stable model catalogue with v0.166.0 on 29 October 2026.
