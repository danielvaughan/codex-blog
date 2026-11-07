---
title: "Guardian v3: Production Configuration Guide for Codex CLI"
date: 2026-11-06T08:00:00+00:00
last_modified_at: 2026-11-07T01:04:37Z
parent: "Articles"
nav_order: 1220
tags:
  - codex-cli
  - guardian-v3
  - safety
  - enterprise
  - configuration
  - auto-review
  - agentic-workflows
  - v0.167
copy_edited: true
---

# Guardian v3: Production Configuration Guide for Codex CLI

**Applies to:** Codex CLI v0.167.0 and later (Guardian v3 feature flag; elastic budgeting in stable)
**Prerequisite reading:** [Guardian v3's Evidence Pipeline: Snapshots, Cursors, and Housekeeping](/2026/11/05/guardian-v3-evidence-pipeline-snapshots-cursors-housekeeping/)

---

Guardian v3 became production-ready with the v0.167.0 stable release on 6 November 2026. The elastic context budgeting changes that shipped in that release — validation of section budgets, whole-entry allocation, and capped pre-allocation — close the last set of reliability gaps that had made Guardian v3 unsuitable for sustained production workloads. The evidence pipeline work (EvidenceSnapshot, EvidenceCursor, Housekeeper, denial reconsideration) merges into the v0.168.0 alpha sprint and will be stable in the next release.

This guide covers the full configuration surface for Guardian v3: how to enable it, what knobs exist, what the enterprise managed-config layer provides, and how to express Guardian v3 constraints in `AGENTS.md`.

It assumes you are already running Codex CLI with auto-review enabled under Guardian v2. If you are not, start with the existing [auto-review configuration reference](#) first.

---

## 1. Enabling Guardian v3

Guardian v3 is feature-flagged and disabled by default. You enable it in `config.toml` or in a managed `requirements.toml`:

```toml
# config.toml (local or per-profile)
[features]
guardianv3 = true
```

```toml
# requirements.toml (managed, enterprise-pushed)
[features]
guardianv3 = { enabled = true }
```

Both forms are accepted: a bare boolean or a table with an optional `enabled` field. This matches the standard Codex feature-flag convention.

**What enabling v3 changes:**

| Aspect | v2 behaviour | v3 behaviour |
|--------|-------------|-------------|
| Evidence delivery | Live history snapshot, single-pass | Immutable EvidenceSnapshot + EvidenceCursor (incremental ACK) |
| Reviewer blocking | Review may block main runtime | Main runtime never blocked; evidence is decoupled via message board |
| Context budgeting | Fixed allocation | Elastic allocation with entry-level caps and adaptive serving assessments |
| Forked/reverted sessions | Reviewer evaluates whatever is available | `EvidenceSnapshot::capture` rejects forked or reverted histories |
| Denial recovery | Terminal denials are final | Optional retry with original inference request (see §3) |

Guardian v3 requires the OpenAI provider. The `GuardianNativeMultiAgentHistory` feature flag and native Guardian prompt construction are explicitly gated on both the `guardianv3` flag and an OpenAI provider ID check. Bedrock and LiteLLM sessions are excluded from native multi-agent history.

---

## 2. The Core Auto-Review Config (Compatible with v3)

Guardian v3 does not change the top-level approval routing configuration. Everything that worked with v2 continues to work. The config below is the starting point for any Guardian v3 production deployment:

```toml
# config.toml
approval_policy = "on-request"
approvals_reviewer = "auto_review"

[auto_review]
policy = """
Deny any action that:
- exfiltrates credentials or secrets
- weakens sandbox policy persistently
- executes commands outside the declared workspace
- makes network requests to undeclared endpoints
Default to approve for read operations and in-workspace edits.
"""
timeout_ms = 90000  # 90 seconds (default)
```

`auto_review.policy` is the plain-text instruction set given to the Guardian reviewer. It can reference your project's security posture, declare trusted domains, and specify escalation conditions. For enterprise deployments this is typically overridden by `guardian_policy_config` in managed requirements (see §5).

---

## 3. Review Timeout Configuration

The Guardian review timeout is the most practically important knob for teams with non-default deployment topologies.

```toml
[auto_review]
timeout_ms = 90000   # 90 seconds (default) — suitable for hosted OpenAI
# timeout_ms = 30000   # 30 seconds — low-latency CI pipelines
# timeout_ms = 180000  # 3 minutes — high-latency enterprise proxies, air-gapped setups
# timeout_ms = 0        # Disable the deadline (preserves cancellation and retry limits)
```

Setting `timeout_ms = 0` is **not the same as disabling Guardian**. It disables only the wall-clock deadline. Cancellation (from user interrupt or session teardown), retry attempt limits, and transport-level timeouts all remain active. Use it when your Guardian deployment consistently exceeds 90 seconds for legitimate reviews — typically air-gapped environments routing through a managed proxy with high round-trip latency.

Configured budget is reported in review telemetry as the numeric value (or `0` for unlimited), so your observability pipeline can flag sessions where the timeout was the binding constraint.

---

## 4. Denial Reconsideration

Denial reconsideration is a Guardian v3 feature that adds a single retry path for terminal denials. Enable it via its feature flag:

```toml
[features]
guardianv3 = true
guardian_denial_reconsideration = true
```

**How it works:**

When the Guardian reviewer returns a terminal denial — a final "do not proceed" verdict — the default behaviour has always been: action blocked, session continues without that action. With denial reconsideration enabled:

1. The first terminal denial triggers a **single retry** using the exact original inference request (not a reformulated version).
2. The model's provisional output for the denied action is **buffered** while the retry runs. If the retry approves, the buffer is released. If it denies again or times out, the original denial stands and the buffer is discarded.
3. The old `experimental_denial_reconsideration_prompt` configuration key is **removed** in v3. The retry no longer uses a custom prompt — it uses the original request directly. This eliminates a prompt-injection surface on the reconsideration path.

The intent is narrow: catch denials produced under transient degraded conditions (timeout, partial context, resource-constrained reviewer) rather than catch genuine security decisions. One retry with the original request is enough to distinguish transient from deliberate.

**When to enable it:**

| Scenario | Recommendation |
|----------|----------------|
| Team has reliable Guardian deployment, low timeout-induced denials | Leave disabled; denials are intentional |
| High-latency or air-gapped deployment; occasional timeout denials | Enable; pair with `timeout_ms = 0` |
| CI pipeline with strict security posture | Leave disabled; any denial should be treated as a build failure |
| Exploratory / developer workstation use | Enable; reduces friction from reviewer transients |

---

## 5. Elastic Budgeting (v0.167.0)

The elastic context budgeting changes in v0.167.0 require no additional configuration — they are automatically applied when Guardian v3 is enabled. The changes are:

- **Validated elastic section budgets** — section-level token allocations are validated against the model's context limit before review begins. Invalid budgets fail fast rather than producing truncated or malformed review context.
- **Whole-entry elastic allocation** — entries that do not fit in the remaining budget are excluded in full rather than being truncated mid-entry. The reviewer always receives complete entries or nothing.
- **Capped pre-allocation** — entries are capped before the elastic allocation pass, preventing any single large entry from consuming the entire context budget.
- **Shared context delivery token limit** — the token limit for context delivery is shared across the review session rather than applied per-entry, enabling better packing for multi-turn reviews.
- **Adaptive serving assessments** — Guardian uses lighter-weight serving assessments (rather than full review) for operations it has high confidence about, reducing latency for low-risk tool calls without lowering coverage.

The net effect is that Guardian v3 review quality is now predictable across session lengths. Long-running overnight agents that previously triggered low-quality reviews (due to context overflow or truncated entries) now receive properly budgeted review context.

If you observe Guardian spending more tokens than expected after upgrading, this is typically the capping and elastic allocation surfacing review budget that was previously silently lost to truncation. The total review quality is higher; the per-review cost may be slightly higher as well.

---

## 6. Granular Approval Policy Surfaces

Guardian v3 is compatible with the full granular approval surface introduced in v0.125+. Configure which event types Guardian reviews:

```toml
[approval_policy.granular]
sandbox_approval   = true    # Sandbox escalation prompts → Guardian reviews
mcp_elicitations   = true    # MCP server prompts → Guardian reviews
rules              = true    # Execpolicy rule triggers → Guardian reviews
skill_approval     = false   # Skill-script prompts → auto-rejected
request_permissions = false  # Permission-tool prompts → auto-rejected
```

**Recommended production starting point:**

```toml
# High-coverage Guardian v3 — reviews sandbox, MCP, and policy triggers;
# auto-rejects skill and permission prompts for throughput.
[approval_policy.granular]
sandbox_approval   = true
mcp_elicitations   = true
rules              = true
skill_approval     = false
request_permissions = false
```

For CI pipelines where all approval prompts should be treated as failures:

```toml
approval_policy = "never"
# Guardian not needed in a fully pre-validated pipeline.
```

---

## 7. Guardian v3 and AGENTS.md

`AGENTS.md` is the per-repository governance layer for Codex CLI. Guardian v3 interacts with it in two ways.

### Managed Network Permission Constraints (v0.167.0)

The `[permissions.network]` block now supports managed network requirement constraints, allowing repositories to declare required network access controls that Guardian enforces at session start:

```markdown
<!-- AGENTS.md -->
## Codex CLI Network Policy

This repository's Codex sessions run with restricted network access.

[permissions.network]
allowed = ["api.openai.com", "registry.npmjs.org", "github.com"]
prompt-user = ["*.internal.corp"]
deny = ["*"]
```

The `prompt-user` directive requires fresh human approval at runtime for matching endpoints rather than relying on cached policy decisions. This is relevant to repositories with compliance requirements around network access — the declaration is auditable and enforced at the AGENTS.md layer rather than requiring out-of-band tooling.

### Guardian Review Guidance in AGENTS.md

Guardian v3's reviewer reads AGENTS.md as part of its context. You can include explicit guidance for the reviewer:

```markdown
<!-- AGENTS.md -->
## Guardian Review Guidance

The Guardian reviewer should:
- Approve all read-only file operations in this repository
- Require sandbox escalation approval for any `npm publish`, `git push --force`, or deployment commands
- Treat any network request outside `allowed` endpoints as high-risk
- Deny any credential read operation that is not for the project's declared service accounts

This project's trusted package registries: registry.npmjs.org, pkg.github.com
```

Because Guardian v3's EvidenceSnapshot includes the AGENTS.md content as repository context, this guidance is reliably in scope for every review — it is part of the immutable snapshot, not injected on the fly.

---

## 8. Enterprise Managed Configuration

For teams deploying Codex CLI at scale, Guardian v3 policy is best managed through the managed requirements layer:

```toml
# requirements.toml (admin-managed, deployed to workstations via your MDM or secrets manager)
[features]
guardianv3 = true
guardian_denial_reconsideration = false   # Set true for high-latency deployments

[auto_review]
timeout_ms = 90000

guardian_policy_config = """
Organisation: Acme Corp
Approved execution environments: CI agents, developer workstations with verified identity
Deny: credential exfiltration, persistent sandbox weakening, undeclared external network requests
Require escalation: any deployment command, any infrastructure change, any IAM operation
Trusted domains: *.acme.internal, api.openai.com, registry.npmjs.org
"""

allowed_approvals_reviewers = ["auto_review"]
```

`guardian_policy_config` overrides `auto_review.policy` for all sessions on the managed device. `allowed_approvals_reviewers` constrains which reviewer integrations users can select — preventing users from bypassing Guardian by switching to `approvals_reviewer = "user"`.

---

## 9. Observability

Guardian v3 review events appear in the standard Codex CLI telemetry stream. With the tracing expansion shipped in v0.165.0, review spans carry:

- `turn_number` — which turn triggered the review
- `tool_name` — which tool call or approval type was reviewed
- `skill_name` — if the action originated from a skill invocation
- `review_outcome` — `approved`, `denied`, or `timed_out`
- `timeout_ms_configured` — the configured budget (or `0` for unlimited)
- `serving_assessment` — `true` if an adaptive serving assessment was used rather than full review

Feed these spans to your OpenTelemetry-compatible backend (Honeycomb, Datadog, Jaeger) to track:
- Review latency by tool type and turn position
- Denial rate by policy rule and session context
- Serving assessment coverage (what fraction of reviews are lightweight vs. full)
- Timeout-induced denial rate (indicator that `timeout_ms` needs adjusting)

---

## Migration from Guardian v2

If you are running Guardian v2 (`guardianv3 = false` or the feature flag absent):

1. **Add the feature flag** to your config or requirements.toml (§1 above).
2. **Review your timeout setting.** v3's elastic budgeting means reviews now consume context more predictably. If you had raised `timeout_ms` to compensate for v2 context overflow, you may be able to reduce it.
3. **Remove `experimental_denial_reconsideration_prompt`** if present — the key is removed in v3. Decide whether to enable `guardian_denial_reconsideration` (§4).
4. **Check your AGENTS.md** for any Guardian-specific guidance sections. They remain valid under v3 but can now include the `[permissions.network]` block for managed network constraints.
5. **Validate elastic budgeting** in a staging session. Run a long-running agent task and inspect the telemetry for `serving_assessment` coverage and review latency. The first run after migration may show slightly different token usage patterns — this is expected as the elastic allocator calibrates to your session's context profile.

Guardian v2 and v3 are not simultaneously active. Enabling the feature flag switches the full session to v3; there is no per-session toggle.

---

## Configuration Reference Summary

| Key | Type | Default | Purpose |
|-----|------|---------|---------|
| `features.guardianv3` | bool or table | `false` | Enable Guardian v3 |
| `features.guardian_denial_reconsideration` | bool | `false` | Enable single retry on terminal denial |
| `features.GuardianNativeMultiAgentHistory` | bool | — | Native multi-agent history (OpenAI only) |
| `approvals_reviewer` | enum | `user` | `auto_review` to route through Guardian |
| `approval_policy` | enum or table | — | `on-request` for standard; `never` for CI |
| `auto_review.timeout_ms` | int | `90000` | Review deadline; `0` = no deadline |
| `auto_review.policy` | string | — | Local review instructions |
| `guardian_policy_config` | string | — | Managed policy (overrides local) |
| `allowed_approvals_reviewers` | list | — | Restrict reviewer selection |
| `[approval_policy.granular].*` | bool | — | Per-surface approval routing |
| `[permissions.network]` in AGENTS.md | block | — | Managed network constraints |
