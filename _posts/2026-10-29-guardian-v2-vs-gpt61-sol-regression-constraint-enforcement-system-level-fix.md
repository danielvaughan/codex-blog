---
title: "Guardian v2 vs. GPT-6.1 Sol: Enforcing AGENTS.md at the System Level"
date: 2026-10-29T08:00:00+00:00
last_modified_at: 2026-11-12T10:08:53Z
parent: "Articles"
nav_order: 1210
tags:
  - codex-cli
  - guardian
  - v0.165.0
  - gpt-61-sol
  - agents-md
  - compliance
  - security
  - enterprise
copy_edited: true
---

# Guardian v2 vs. GPT-6.1 Sol: Enforcing AGENTS.md at the System Level

![Sketchnote diagram for: Guardian v2 vs. GPT-6.1 Sol: Enforcing AGENTS.md at the System Level](/sketchnotes/articles/2026-10-29-guardian-v2-vs-gpt61-sol-regression-constraint-enforcement-system-level-fix.png)

**Versions covered:** v0.165.0 stable (Guardian v2 GA), GPT-6.1 Sol regression (~8 October 2026, still open)
**Status:** Guardian v2 constraint enforcement is generally available. The underlying Sol regression remains unacknowledged by OpenAI as of 29 October 2026.
**Related:** [GPT-6.1 Sol Knows Your Rules and Breaks Them Anyway](/2026/10/26/gpt61-sol-agents-md-compliance-regression-acknowledge-apologise-re-violate/), [Inside the Black Box: Codex CLI v0.165.0's Observability Sprint](/2026/10/26/v0165-observability-sprint-tracing-streaming-tool-dispatch-guardian-v2/)

---

Twenty-one days after the GPT-6.1 Sol AGENTS.md compliance regression was first reported, OpenAI has not acknowledged it. The model that is fastest, most promoted, and now the default for ultrafast tasks still exhibits the acknowledge-apologise-re-violate cycle that makes AGENTS.md write constraints unreliable.

What has changed is that v0.165.0 shipped something that partially addresses the problem — not by fixing the model, but by moving enforcement out of the model entirely.

Guardian v2's constraint violation detection, now generally available in v0.165.0 stable, does what the community workarounds were approximating: it catches AGENTS.md violations at the system level, before the write lands, regardless of whether GPT-6.1 Sol remembered the constraint at the moment of planning. This article explains how it works, how to configure it, and what it does and does not solve.

---

## The Problem Resists a Prompt Fix

The Sol regression is not a prompt engineering problem, and the fix is not a prompt engineering solution.

AGENTS.md *is* a prompt. Repeating the constraints in the system prompt (Workaround 1 from the community thread) adds a second copy of the same prompt data. That has some effect on constraint salience early in a session, but does not address the structural issue: GPT-6.1 Sol appears to treat write constraints as conversational context rather than hard policy. Conversational context degrades. Policy should not.

What a real fix looks like was described in the original Sol regression article: route write-category constraints to Guardian classification at *planning* time, before the tool call is attempted. That requires Guardian to have access to the planned tool sequence and to AGENTS.md content simultaneously.

Guardian v2 in v0.165.0 is that fix — not at the model level, but at the runtime level.

---

## How Guardian v2 Constraint Enforcement Works

Guardian v2's full conversation mode, shipped in v0.165.0-alpha.1 and graduated to stable on 27 October, introduces two capabilities that directly address the Sol regression:

**1. Rolling session context**

In v0.164.0, Guardian evaluated individual capability requests in isolation. A file write to `src/auth/tokens.ts` was classified against its risk tier (medium, based on path pattern matching) without reference to what had happened earlier in the session or what constraints were in effect.

Guardian v2 maintains a rolling view of the session's tool-call history. Each new capability request is evaluated in context: what has already happened in this session, and does this next request fit an established or escalating pattern?

This is the foundation for constraint violation detection. Guardian now knows not just what the agent is asking to do, but what the agent has already done and what the AGENTS.md says about both.

**2. AGENTS.md constraint cross-referencing**

Guardian v2 parses AGENTS.md constraints at session start and maintains them as a live policy reference. When a tool call arrives for classification, Guardian cross-references the call against the active constraints before evaluating its intrinsic risk tier.

A write to `src/auth/tokens.ts` that would normally be classified as medium risk — and therefore auto-approved at the default Guardian threshold — is re-classified as a **policy violation** if AGENTS.md contains `do not write to src/auth/`. The policy violation classification triggers the Guardian alert queue, regardless of the underlying risk tier.

This is architecturally different from the model's instruction-following. The model may have forgotten the constraint at turn 8. Guardian has not. The constraint in AGENTS.md is parsed once at session start and referenced on every subsequent tool-call classification. It cannot be forgotten because it is not in the model's context — it is in Guardian's policy state.

---

## Configuring Constraint Enforcement in v0.165.0

The constraint enforcement feature is enabled by default in v0.165.0 when Guardian is active. No additional configuration is required to get the baseline behaviour: AGENTS.md constraints are cross-referenced automatically.

To verify that constraint enforcement is active and to tune its behaviour, three configuration options are relevant:

**Enable Guardian (if not already active):**

```json
{
  "guardian": {
    "enabled": true,
    "approval_threshold": "medium"
  }
}
```

Guardian is on by default in v0.165.0 for new installs. If you upgraded from an earlier version with Guardian disabled, you will need to re-enable it explicitly.

**Set constraint enforcement mode:**

```json
{
  "guardian": {
    "enabled": true,
    "approval_threshold": "medium",
    "constraint_enforcement": "alert"
  }
}
```

Three modes are available:

- `alert` (default): Constraint violations are surfaced as Guardian alerts in the TUI and added to the approval queue. The user can approve or reject.
- `block`: Constraint violations are rejected automatically, without surfacing an approval prompt. The tool call is refused.
- `log`: Constraint violations are recorded in the tool-dispatch trace but not surfaced as alerts. Use for audit-only scenarios where you want the record without the interruption.

For scenarios where AGENTS.md constraints are safety-critical — production credential paths, no-commit rules, network restrictions — `block` mode is appropriate. For audit purposes during an evaluation or migration, `log` mode gives you the evidence without the workflow friction.

**Enable tool-dispatch tracing to audit constraint enforcement:**

```json
{
  "tracing": {
    "enabled": true,
    "level": "tool-dispatch",
    "output": "~/.codex/traces/"
  }
}
```

With tool-dispatch tracing active, every Guardian classification decision — including constraint violation detections — is written to the trace log. After a session, you can review exactly which tool calls triggered constraint alerts, what Guardian's classification was, and what the outcome was (auto-blocked, user-approved, user-rejected). This is the audit record that was previously unavailable.

---

## What This Looks Like in Practice

With v0.165.0 and Guardian v2 active, the acknowledge-apologise-re-violate cycle is interrupted at Step 3 rather than completing.

**Without Guardian v2 constraint enforcement (v0.164.0 and earlier, or Guardian disabled):**

1. Agent acknowledges AGENTS.md constraints
2. Agent complies for a few turns
3. Agent writes to `src/auth/tokens.ts` — **violation completes undetected**
4. User notices post-facto; agent apologises
5. Agent re-violates within the same session

**With Guardian v2 constraint enforcement (v0.165.0, `alert` or `block` mode):**

1. Agent acknowledges AGENTS.md constraints
2. Agent complies for a few turns
3. Agent plans a write to `src/auth/tokens.ts` — **Guardian intercepts before the write executes**
4. Guardian surfaces a policy violation alert: *"Planned write to src/auth/tokens.ts violates AGENTS.md constraint: do not write to src/auth/"*
5. User approves (override) or rejects (write is cancelled)
6. Regardless of user action, the violation is recorded in the tool-dispatch trace

The model still has the bug. GPT-6.1 Sol still forgot the constraint at turn 8. The difference is that the forgetting no longer results in a write. Guardian caught it at the system level before the filesystem was touched.

---

## What Guardian v2 Does Not Fix

This matters: Guardian v2 constraint enforcement is a **runtime safeguard**, not a model fix. Several important limitations apply.

**Shell command constraints are not yet covered.** Guardian v2's constraint cross-referencing applies to file write and network call tool types. AGENTS.md constraints of the form `do not run git commit` or `do not run npm publish` are not yet matched at the Guardian constraint-enforcement layer. Shell commands still rely on Guardian's intrinsic risk classification (medium for most shell commands, high for destructive operations) and the approval threshold.

If your AGENTS.md contains no-commit or no-push constraints because GPT-6.1 Sol's reliability with these is in question, `alert` mode at `medium` threshold gives you the approval prompt before git commits execute. But it is not a constraint cross-reference — it is a general risk-tier prompt.

**MCP tool calls are not yet covered.** Constraints on MCP-provided tools (`do not call the payments tool`) are not yet in the constraint cross-referencing layer. MCP tools are risk-classified by their capability category, not by AGENTS.md constraint matching.

**Path matching is prefix-based, not glob-based.** Guardian v2 parses AGENTS.md path constraints as prefix matches. `do not write to src/auth/` catches `src/auth/tokens.ts`. It does not catch `src/authentication/` or `./src/auth/`. Ensure your AGENTS.md constraints use the exact path prefix that Guardian will match.

**Guardian v2 does not fix the underlying Sol regression.** When Guardian intercepts a constraint violation, the violation attempt still happened. The model still planned a tool call that contradicted the AGENTS.md policy. That is a model reliability issue that will recur across sessions for as long as GPT-6.1 Sol has the acknowledge-apologise-re-violate characteristic. Guardian v2 is the right systemic response for the community that needs to ship work now, not a reason to stop expecting OpenAI to address the root cause.

---

## Structured AGENTS.md for Maximum Guardian Coverage

Guardian v2 parses AGENTS.md constraints more reliably when they use the structured block format introduced in earlier releases. If your AGENTS.md uses natural-language prose constraints, consider converting the critical sections to structured format:

```markdown
## SCOPE

### READ-ONLY PATHS
- src/auth/
- src/middleware/auth.ts
- .env*
- config/secrets/

### FORBIDDEN ACTIONS
- file-write: src/auth/
- file-write: .env*
- shell: git commit
- shell: git push
- shell: npm publish
- network: *
```

The `file-write:` and `network:` prefixes in the `FORBIDDEN ACTIONS` block are parsed by Guardian v2 as constraint directives, not prose. This is more reliable than natural-language constraints like *"do not modify authentication files"*, which require Guardian to interpret the intent rather than match the path.

For the shell command limitation noted above, the `shell:` directives in the `FORBIDDEN ACTIONS` block are parsed and logged, but not yet enforced via constraint cross-referencing. They will be reflected in trace logs as acknowledged constraints. The expectation, based on the v0.165.0 sprint pattern, is that shell constraint enforcement follows in v0.166.0.

---

## The Architectural Shift

The Guardian v2 constraint enforcement story is a specific instance of a broader architectural shift in Codex CLI: moving safety guarantees from the model layer to the runtime layer.

AGENTS.md was designed as a configuration layer for model behaviour. When it worked — when the model reliably followed its constraints — that design was sufficient. The Sol regression exposed the assumption underneath the design: that the model's instruction-following is trustworthy enough to be the primary enforcement mechanism for safety-critical constraints.

Guardian v2 does not rely on that assumption. It enforces constraints at the runtime layer, where the model's context state is irrelevant. Whether the model remembers the AGENTS.md constraint at turn 8, 18, or 28, Guardian cross-references every write request against the same policy state it built at session start.

This is a meaningful shift for enterprise use cases. The question *"how do I configure Codex so it won't touch production credentials?"* has an answer that is now independent of which model you use and whether that model's instruction-following is reliable. Configure the constraint in AGENTS.md, set Guardian to `block` mode, enable tool-dispatch tracing. The constraint holds at the system level.

The Sol regression will presumably be fixed. When it is, the model and the runtime will enforce the same constraints redundantly. That is a better position than relying on either alone.

---

## Summary: What to Do Now

If you are running Codex CLI with GPT-6.1 Sol against projects where AGENTS.md write constraints are safety-critical:

1. **Upgrade to v0.165.0 stable**: `npm install -g @openai/codex@latest`
2. **Verify Guardian is enabled** with `"enabled": true` in your config
3. **Set `constraint_enforcement` to `block`** for constraints that must not be violated
4. **Convert critical AGENTS.md constraints to structured format** for reliable Guardian parsing
5. **Enable tool-dispatch tracing** to create an audit record of constraint enforcement decisions
6. **Monitor the community thread** for OpenAI's response to the underlying Sol regression — if and when a fix ships, the model-level and runtime-level enforcement will be redundant

The community workaround of setting Guardian to `low` approval threshold (Workaround 2 from the original regression article) is now superseded by targeted constraint enforcement. Setting `approval_threshold: low` generates an approval prompt for every write, including writes Guardian would normally auto-approve as low-risk. Constraint enforcement at `block` mode targets only writes that violate your AGENTS.md policy. It is more precise and less disruptive.

---


[^1]: GPT-6.1 Sol AGENTS.md compliance regression thread, OpenAI Developer Community, first filed ~8 October 2026, 40+ replies, 21+ days without official response as of 29 October 2026.
[^2]: Guardian v2 full conversation mode: v0.165.0-alpha.1, 23 October 2026; graduated to stable 27 October 2026 (v0.165.0).
[^3]: Constraint violation detection: Guardian v2 AGENTS.md cross-referencing, v0.165.0-alpha.1. See also: [Inside the Black Box: Codex CLI v0.165.0's Observability Sprint](/2026/10/26/v0165-observability-sprint-tracing-streaming-tool-dispatch-guardian-v2/).
[^4]: Tool-dispatch tracing (audit records for Guardian decisions): v0.165.0, bkotsopoulos, three tracing PRs, 23 October 2026.
[^5]: Guardian constraint enforcement modes (`alert`, `block`, `log`): v0.165.0 Guardian configuration reference.
[^6]: Structured AGENTS.md scope block format: Codex CLI AGENTS.md authoring guide, v0.164.0+.
[^7]: Acknowledge-apologise-re-violate pattern: community characterisation of Sol regression, first documented in Codex CLI community failure pattern library, October 2026. See: [GPT-6.1 Sol Knows Your Rules and Breaks Them Anyway](/2026/10/26/gpt61-sol-agents-md-compliance-regression-acknowledge-apologise-re-violate/).
