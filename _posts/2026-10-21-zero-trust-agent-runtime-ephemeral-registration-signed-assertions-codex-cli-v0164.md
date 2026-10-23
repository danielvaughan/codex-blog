---
title: "Zero-Trust Agent Runtime: Ephemeral Registration and Signed Assertions in Codex CLI v0.164.0"
date: 2026-10-21T09:00:00+00:00
last_modified_at: 2026-10-23T11:06:20+01:00
parent: "Articles"
nav_order: 1193
tags:
  - codex-cli
  - v0.164.0
  - security
  - zero-trust
  - enterprise
  - guardian
  - authentication
copy_edited: true
---

# Zero-Trust Agent Runtime: Ephemeral Registration and Signed Assertions in Codex CLI v0.164.0

![Sketchnote diagram for: Zero-Trust Agent Runtime: Ephemeral Registration and Signed Assertions in Codex CLI v0.164.0](/sketchnotes/articles/2026-10-21-zero-trust-agent-runtime-ephemeral-registration-signed-assertions-codex-cli-v0164.png)

**Release track:** v0.164.0-alpha.4, alpha.5, alpha.6 (19 October 2026)
**Stable track:** v0.163.0 (current stable, 15 October 2026)
**Impact:** Enterprise deployments, regulated environments, teams running parallel agentic tasks

---

Three alphas landed on 19 October 2026 — alpha.4, alpha.5, and alpha.6 — all focused on a single theme: hardening the security of the Codex agent runtime. The changes introduced a zero-trust security model for agent capability requests, built on two new primitives: ephemeral registration and signed assertions.

These changes sit below the user-visible layer. You will not see a new command or a new UI element. What you will see, in environments where they matter, is a Codex CLI that can be deployed into zero-trust networks, audited for capability use, and integrated with enterprise identity systems — without compromises or custom workarounds.

---

## The Problem: Agent Identity Is Ambiguous at Runtime

Consider how Codex CLI runs in a typical enterprise deployment before v0.164.0. A developer authenticates with `codex auth`, establishing their identity. When they run a Codex task, the task executes under that developer's identity. If the task spawns sub-agents — for example, a Guardian subagent reviewing a file write — those sub-agents run under the same identity as the parent.

This model has a correctness problem: capability requests from a sub-agent look identical to capability requests from the user. If the Guardian subagent requests a file read, and the user directly requests a file read, they are indistinguishable from an audit perspective. This matters in two ways:

1. **Audit trails are incomplete.** A log entry showing "file read: src/auth/tokens.ts" does not tell you whether that read was user-initiated or agent-initiated, nor which agent in the tree initiated it.

2. **Privilege escalation is possible.** A sufficiently capable agent, or a compromised MCP tool, could request capabilities using the parent session's full identity without any additional verification. The parent session's capabilities become the effective capability floor for everything it spawns.

Zero-trust agent runtime security addresses both problems.

---

## Ephemeral Registration: One Identity Per Agent Instance

The first primitive is ephemeral registration. When Codex creates a new agent instance — whether a top-level session or a sub-agent spawned during a task — it now performs a registration step that issues a short-lived identity token to that instance.

The registration token is:
- **Short-lived:** it expires at a fixed TTL, typically aligned with the expected task duration
- **Instance-scoped:** it is unique to the specific agent instance, not reusable by other instances
- **Hierarchically bound:** it carries a reference to its parent agent's identity, creating a verifiable chain from sub-agent back to root session

What this means in practice:

When the Guardian subagent requests approval for a file write, it presents its ephemeral registration token alongside the request. Codex's approval mechanism verifies the token before processing the request. The token's parent chain shows: *this request came from Guardian, which was spawned by session X, which is running under user Y's account.*

This chain is logged. Every capability request in a Codex session now has a full provenance trace: user → session → agent instance → capability request. In a regulated environment where an auditor needs to know exactly what happened during an agentic task, this trace is the foundation for that answer.

---

## Signed Assertions: Every Capability Request Is Attested

The second primitive is signed assertions. Alongside the ephemeral registration token, every capability request from an agent instance now carries a cryptographically signed assertion of intent.

The assertion is a small JSON structure:

```json
{
  "agent_id": "guardian-sub-7f2a",
  "parent_id": "session-codex-abc123",
  "capability": "file.write",
  "target": "src/auth/tokens.ts",
  "justification": "Updating token TTL as specified in task instructions",
  "timestamp": "2026-10-19T14:32:07Z",
  "signature": "..."
}
```

The signature is computed using the agent instance's ephemeral key (issued at registration). Codex's approval and logging infrastructure verifies the signature before processing the capability request.

The `justification` field is populated by the agent itself — it is the agent's natural-language statement of why it needs the capability. This is not a security control (an agent can write anything in justification), but it serves two purposes:

1. **Human-in-the-loop approval:** When a capability request is sent to a human approver, the justification gives the approver context without requiring them to read the full session history.
2. **Post-hoc audit:** The justification, preserved in the audit log alongside the signature, provides a human-readable narrative of agent decision-making.

---

## Guardian Hardening: Stricter Classification, Tighter Defaults

The Guardian subagent — Codex CLI's built-in risk-classification layer — received targeted hardening in alpha.4–6.

Guardian classifies capability requests into risk tiers (low, medium, high, critical) and determines whether each request can proceed automatically or requires approval. Before the alpha.4–6 hardening:

- The classification thresholds were calibrated for productivity (minimise approval interruptions)
- Some capability combinations that individually were low-risk could collectively constitute a higher-risk pattern without triggering a higher classification
- Guardian did not have access to the agent provenance chain when making classifications

Post-hardening:

- Guardian receives the ephemeral registration token and parent chain as part of every classification request
- Guardian's risk model now considers *who is asking* (user vs. sub-agent, and which sub-agent in which context) alongside *what is being requested*
- Compound capability patterns — for example, file read followed by network write to an external endpoint — can now be classified as a higher-risk sequence even if each individual capability would be low-risk in isolation
- The default threshold for automatic approval has been tightened: more requests that previously auto-approved now go to the approval queue

This last change is a deliberate trade-off. Tighter defaults mean more approval prompts in some workflows. The upside is that the default configuration is now appropriate for regulated and security-sensitive environments without additional configuration. Teams that prefer the previous behaviour can relax the thresholds in their Codex configuration.

---

## Async Question Markdown Prompts

A UX improvement that arrived in the same alpha batch: approval questions from the Guardian subagent are now rendered as Markdown in the Codex TUI.

Previously, approval questions were presented as plain text. In practice, this meant questions about complex capability requests — involving file paths, code snippets, or structured data — were difficult to read. A question like "May I write the following content to tokens.ts?" followed by 40 lines of code in a monospaced plain-text block was not a comfortable reading experience.

With async question Markdown prompts:
- Code blocks are syntax-highlighted
- File paths are formatted as inline code
- Structured data (JSON, YAML) renders with appropriate indentation
- Questions are presented asynchronously — if multiple sub-agents are waiting for approval simultaneously, each question queues and the user processes them in sequence without blocking the other agents

The async behaviour is particularly significant for parallel multi-agent workflows. Previously, an approval request from any agent would block the entire session until the user responded. With async queuing, parallel agents continue running while one waits for approval, and the user sees a clear queue of pending decisions when they return to the terminal.

---

## Practical Implications for Enterprise Deployment

### Audit and compliance

If your organisation's security team has asked whether Codex CLI can be deployed in a way that produces auditable records of agent capability use, the answer after v0.164.0-alpha.4–6 is yes, with conditions. The ephemeral registration chain and signed assertions provide the raw material for a compliance-grade audit log. What is not yet provided out of the box is a log export format or integration with common SIEM tools — that work will likely arrive in later alphas or in the stable release.

### Zero-trust network requirements

Ephemeral registration relies on a lightweight token issuance service. In v0.164.0 alphas, this runs locally within the Codex process — there is no external network dependency for registration. This means the feature works in air-gapped environments and does not require any additional firewall rules or outbound network allowances.

### Privilege boundaries for multi-agent tasks

The parent chain model provides the technical foundation for privilege scoping: a sub-agent's effective capabilities could be constrained to a subset of the parent session's capabilities. The alpha implementation does not yet expose this as a user-configurable policy, but the data structure supports it. Teams that need sub-agent privilege scoping will likely see this arrive in a later alpha.

---

## What Is Still Missing

The alpha.4–6 security work is foundational — it lays the plumbing on which more visible security features can be built. What is not yet present in v0.164.0 alphas:

- **Policy-as-code for capability grants:** No declarative configuration format for defining per-project capability policies
- **Sub-agent privilege scoping:** Parent chain is tracked but not yet used to restrict sub-agent capabilities
- **SIEM integration / log export:** Signed assertions are logged internally but not yet exportable in standard formats
- **Cross-session attestation:** Ephemeral tokens are instance-scoped and do not currently chain across sessions (relevant for ctx-kit and estafeta-style pipeline handoffs)

These gaps suggest where the security feature set will grow through the v0.164.0 release cycle and into v0.165.0.

---

## Upgrading to Test the Alpha

```bash
npm install -g @openai/codex@0.164.0-alpha.8
```

Note: alpha.4–6 security changes are included in alpha.7 and alpha.8, which also carry the enterprise MCP auth overhaul. There is no reason to install an earlier alpha.

Guardian hardening defaults are enabled automatically. If you find the increased approval frequency disruptive in local development, you can adjust the threshold in your `~/.codex/config.json`:

```json
{
  "guardian": {
    "approval_threshold": "medium"
  }
}
```

`"medium"` restores the pre-hardening behaviour for medium-risk requests. `"high"` only requires approval for high and critical requests.

---

## v0.163.0 Caveats

As with the MCP OAuth changes, the zero-trust security work is in the v0.164.0 alpha track only. v0.163.0 stable is unaffected by these changes and retains the previous Guardian model. v0.163.0 has three known platform regressions (Windows 11 26H2 daemon failure, Ubuntu 24.04 hang, macOS compaction) — see the [v0.163.0 stable release article](../2026-10-16-codex-v0163-stable-release-config-follow-up-parallel-tools-daybreak/) for details and the v0.162.1 downgrade workaround.

---

## Summary

The v0.164.0-alpha.4–6 zero-trust agent runtime work introduces two new security primitives — ephemeral registration and signed assertions — that transform Codex CLI from a single-identity agent into a traceable, auditable, hierarchically-attributed system. Guardian hardening tightens the default approval model to be safe-by-default. Async Markdown approval prompts make human-in-the-loop review manageable in parallel multi-agent workflows.

These are not features that make Codex faster or more capable in the everyday sense. They are the features that make it deployable in environments where "we have no idea what the agent did" is not an acceptable answer.

---


[^1]: v0.164.0-alpha.4, alpha.5, alpha.6 — ephemeral registration, signed assertions, Guardian hardening. OpenAI Codex CLI repository, 19 October 2026.
[^2]: Ephemeral agent registration and signed assertions described via commit metadata, PR commentary, and community documentation, October 2026.
[^3]: Guardian subagent hardening: stricter classification thresholds, provenance-aware risk model. v0.164.0-alpha.4.
[^4]: Async question Markdown prompts: v0.164.0-alpha.5. Human-in-the-loop approval queue for parallel agents.
[^5]: v0.163.0 platform regressions: community reports, 15–18 October 2026.
