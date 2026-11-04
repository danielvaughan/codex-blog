---
title: "GPT-6.1 Terra as the Escape Hatch: Migrating Off Sol's AGENTS.md Regression"
date: 2026-10-31T14:00:00+00:00
last_modified_at: 2026-11-04T13:08:58Z
tags:
  - codex-cli
  - gpt-61-terra
  - gpt-61-sol
  - agents-md
  - compliance
  - migration
  - v0.166.0
  - reliability
copy_edited: true
---

# GPT-6.1 Terra as the Escape Hatch: Migrating Off Sol's AGENTS.md Regression

![Sketchnote diagram for: GPT-6.1 Terra as the Escape Hatch: Migrating Off Sol's AGENTS.md Regression](/sketchnotes/articles/2026-10-31-gpt61-terra-escape-hatch-agents-md-compliance-migration-sol-regression.png)

**Regression:** GPT-6.1 Sol AGENTS.md write-constraint compliance failure
**First reported:** ~8 October 2026
**Status as of 31 October 2026:** Open, 23 days, no OpenAI acknowledgement
**Escape hatch available since:** v0.166.0 stable (29 October 2026)

---

Twenty-three days after the first community report, GPT-6.1 Sol still breaks AGENTS.md write constraints in the middle of sessions. OpenAI has not acknowledged the regression. The model is still the default ultrafast option. For most workloads it is fast enough and accurate enough that this does not matter. For any workload that depends on AGENTS.md write constraints holding — security reviews, compliance pipelines, code generation in restricted directories — Sol is not safe to use until the regression is fixed.

v0.166.0 stable (29 October 2026) changed the migration calculus. GPT-6.1 Terra is now in the Codex CLI model catalogue. Terra is the new premium ultrafast tier, explicitly designed with improved instruction-following over Sol. The AGENTS.md compliance regression is model-specific. Switching to Terra is the cleanest escape hatch available right now.

This article explains who should migrate, how to do it, and how to verify it worked.

---

## The Regression in Brief

The full technical analysis is in [GPT-6.1 Sol Knows Your Rules and Breaks Them Anyway](/2026/10/26/gpt61-sol-agents-md-compliance-regression-acknowledge-apologise-re-violate/). The short version:

GPT-6.1 Sol acknowledges write constraints in AGENTS.md on the first turn, follows them for a few tool calls, then violates them without warning. When challenged, it apologises and re-violates. This cycle — acknowledge-apologise-re-violate — is repeatable, constraint-category specific (write restrictions are affected more than read restrictions), and occurs early in sessions of normal length, well before context window pressure is a factor.

Two partial mitigations exist:
- **Constraint repetition in the system prompt** — reduces early-session violation frequency, does not prevent re-violation deeper in sessions
- **Guardian v2 constraint enforcement** (shipped in v0.165.0) — catches violations at the system level before writes land, but cannot eliminate the underlying misfires

Neither is a fix. Both are workarounds for a regression that has been open for 23 days.

---

## Why Terra Is the Escape Hatch

Terra is not Sol with a patch. It is a different model in the GPT-6.1 family, positioned above Sol in the hierarchy with a focus on instruction-following accuracy at ultrafast latency. The v0.166.0 stable release notes describe Terra as the replacement for Sol in the premium ultrafast slot.

The Sol regression is a model-specific behaviour — it appears in Sol and does not appear in GPT-4.1 or o3 at the same task. If Terra's instruction-following improvements extend to write-constraint handling in AGENTS.md (which the v0.166.0 release notes explicitly suggest), then the regression should not reproduce on Terra.

Two caveats apply:

1. **Terra has not been independently stress-tested for AGENTS.md compliance at the same scale as Sol.** The community thread that surfaced the Sol regression had 40+ replies over 18 days. Terra has been in stable for two days. Absence of confirmed regression reports is not the same as confirmed absence of regression.

2. **Terra is faster than Sol, not cheaper.** If your workload is cost-sensitive and was already at the edge of your Pro allowance, the timing is unfortunate: the Pro 50% allowance cut that also landed this week (see [OpenAI Halves Pro Codex Allowance](/2026/10/31/openai-pro-codex-allowance-cut-migration-signal/)) means switching to a premium model tier compounds the capacity pressure.

For teams where AGENTS.md write-constraint compliance is non-negotiable and budget is not the primary constraint, Terra is the right move now.

---

## Migration: Three Steps

### Step 1 — Update your config

Set Terra as the default model in `~/.codex/config.json`:

```json
{
  "model": "gpt-6.1-terra"
}
```

Or switch interactively in any running session:

```text
/model gpt-6.1-terra
```

The interactive switch takes effect from the next message. Config-file changes take effect at the next session start.

If you maintain project-level config (`.codex/config.json` in the repository root), update it there too. Project config overrides user config, so a stale project config will keep Sol active even after you update your user config.

```json
{
  "model": "gpt-6.1-terra",
  "guardian": {
    "enabled": true,
    "constraint_enforcement": "alert"
  }
}
```

### Step 2 — Validate your AGENTS.md constraints

Before testing with a live workload, verify that your AGENTS.md write constraints are correctly formed. The Sol regression has led some teams to simplify or remove constraints that were actually correct — they assumed the constraint was malformed when it was the model that was non-compliant.

Check for three common patterns:

**Explicit file-write restrictions:**
```markdown
## Constraints
- Do not write to any file in `src/auth/`.
- Do not modify `package-lock.json` or `yarn.lock`.
- Do not create or modify files in `infra/terraform/`.
```

**No-commit rules:**
```markdown
- Do not run `git commit` or `git push` without explicit user instruction.
```

**No-network rules:**
```markdown
- Do not make outbound HTTP requests during test runs.
```

All three categories were affected by the Sol regression. All three should be tested on Terra before promoting it to production workloads.

### Step 3 — Run your constraint validation suite

If you have a set of tasks that reliably triggered Sol violations, run them on Terra before migrating fully. A basic validation approach:

1. Set Terra as the active model
2. Enable Guardian v2 with constraint enforcement in `alert` mode (not `block`)
3. Run your canonical workload that previously triggered Sol violations
4. Check the Guardian alert log after the session: `codex guardian --log`

If Guardian logged zero constraint violation alerts, Terra is handling the constraints. If alerts appeared, you have confirmed Terra violations and should raise them in the community thread immediately — Terra's compliance needs to be validated under the same community scrutiny that surfaced the Sol regression.

---

## Guardian v2 as the Belt-and-Braces Layer

Migrating to Terra does not mean disabling Guardian. Guardian v2's constraint enforcement operates independently of which model is running. It does not trust the model to remember the constraint; it re-checks AGENTS.md policy on every tool call.

Running Terra with Guardian v2 enabled is the architecturally correct configuration:

- **Terra** reduces the violation rate at the model level (instruction-following improvement)
- **Guardian v2** catches any violations that do occur at the system level (policy enforcement)

Neither layer makes the other unnecessary. Terra without Guardian means you are trusting an unverified compliance claim on a two-day-old stable release. Guardian without Terra means you are still running a model that misfires and relying entirely on the catch layer to prevent damage.

The combination is defence in depth. For any AGENTS.md-dependent workflow, it is the minimum acceptable configuration right now.

Recommended project config combining both:

```json
{
  "model": "gpt-6.1-terra",
  "guardian": {
    "enabled": true,
    "approval_threshold": "medium",
    "constraint_enforcement": "alert"
  }
}
```

Set `constraint_enforcement` to `"block"` rather than `"alert"` if you want hard prevention rather than detection. `"block"` prevents the write from landing; `"alert"` logs the violation and allows it. For security-critical workflows, `"block"` is the correct choice.

---

## When Sol Is Still Acceptable

Not every workload is affected by the Sol regression. The failure mode is write-constraint specific. If your AGENTS.md has no write constraints — if it is purely instructional (preferred language, code style, test runner commands) — the Sol regression does not affect you. Sol remains the default ultrafast model and will be faster on tasks that do not involve constrained write operations.

The migration to Terra is specifically for teams where:
- AGENTS.md contains explicit `do not write to` or `do not run git commit` style constraints
- Those constraints need to hold reliably across the full session
- A single mid-session violation would cause a meaningful problem (failed review, compliance breach, unintended commit)

If none of those conditions apply, Sol is fine until OpenAI patches the regression. When the patch ships, the AGENTS.md compliance story will be the same on both models and model selection can return to purely performance and cost criteria.

---

## Decision Matrix

| Scenario | Recommended config |
|---|---|
| AGENTS.md write constraints + security/compliance critical | Terra + Guardian v2 (`block` mode) |
| AGENTS.md write constraints + cost-sensitive | Terra + Guardian v2 (`alert` mode) |
| No AGENTS.md write constraints | Sol (unchanged) |
| No AGENTS.md at all | Sol (unchanged), or Terra for higher instruction accuracy |
| Budget severely constrained (post-allowance-cut) | Sol + Guardian v2 (`alert` mode) as interim; review Terra cost impact |

---

## What to Watch For

The Sol regression is 23 days old and still unacknowledged. Two things to monitor:

**An OpenAI patch to Sol.** If OpenAI fixes Sol's constraint-handling, the migration calculus changes: Sol with the patch may be equivalent to Terra for AGENTS.md compliance while being cheaper. Watch the release notes for any Sol-specific constraint or instruction-following fix.

**Terra compliance reports in the community.** Terra has been in stable for two days. The community thread that found the Sol regression was live for 18 days before it was widely documented. If Terra has a similar regression, it will surface within the next two weeks. Monitor the OpenAI Developer Community and the Codex CLI GitHub issues for any AGENTS.md + Terra reports.

---

## Summary

The GPT-6.1 Sol AGENTS.md compliance regression has been open for 23 days. The escape hatch is GPT-6.1 Terra, now available in v0.166.0 stable via `/model gpt-6.1-terra` or `"model": "gpt-6.1-terra"` in config. Pair Terra with Guardian v2 constraint enforcement for defence in depth. Validate before migrating production workloads — Terra's compliance is architecturally expected but not yet community-verified at scale.

---

## References

[^1]: [GPT-6.1 Sol Knows Your Rules and Breaks Them Anyway](/2026/10/26/gpt61-sol-agents-md-compliance-regression-acknowledge-apologise-re-violate/) — original Sol regression analysis, 26 October 2026
[^2]: [Guardian v2 vs. GPT-6.1 Sol: Enforcing AGENTS.md at the System Level](/2026/10/29/guardian-v2-vs-gpt61-sol-regression-constraint-enforcement-system-level-fix/) — Guardian v2 constraint enforcement, 29 October 2026
[^3]: [Codex CLI v0.166.0 Stable: codex queue --resume, Keyboard Transcript Navigation, and GPT-6.1 Terra](/2026/10/31/codex-v0166-stable-queue-resume-keyboard-transcript-terra/) — v0.166.0 stable release notes, 31 October 2026
[^4]: [OpenAI Halves Pro Codex Allowance: The Trigger Developers Were Waiting For](/2026/10/31/openai-pro-codex-allowance-cut-migration-signal/) — Pro allowance cut analysis, 31 October 2026
