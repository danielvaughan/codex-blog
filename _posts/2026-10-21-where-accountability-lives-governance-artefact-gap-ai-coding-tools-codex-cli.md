---
title: "Where Accountability Lives: The Governance Artefact Gap in AI Coding Tools"
date: 2026-10-21T11:00:00+00:00
last_modified_at: 2026-10-21T18:09:22+01:00
parent: "Articles"
nav_order: 1196
tags:
  - codex-cli
  - governance
  - accountability
  - compliance
  - security
  - agents-md
  - hooks
  - enterprise
  - eu-ai-act
  - audit-trail
copy_edited: true
---

# Where Accountability Lives: The Governance Artefact Gap in AI Coding Tools

![Sketchnote diagram for: Where Accountability Lives: The Governance Artefact Gap in AI Coding Tools](/sketchnotes/articles/2026-10-21-where-accountability-lives-governance-artefact-gap-ai-coding-tools-codex-cli.png)

**Research basis:** Lloyd et al. "Where Accountability Lives: Mapping Human Responsibility to Workflow Artifacts in Agentic Software Development" (arXiv:2608.15678, 2026)
**Impact:** Engineering leads, compliance teams, and CISOs deploying AI coding tools at enterprise scale; teams with EU AI Act, SOX, or DORA obligations

---

When something goes wrong with AI-generated code — a production incident, a security vulnerability, a data breach — the immediate question is: who is responsible? The answer, according to a new qualitative study by Lloyd et al., is that the governance infrastructure of every major AI coding tool is structurally unable to answer that question.[^1]

This is not a problem with individual developers' behaviour or a gap in team processes. It is a documented architectural absence in the tools themselves — one that exists independently of how carefully any individual or organisation uses them.

---

## The study

Lloyd et al. conducted a qualitative analysis of 121 governance artefacts across four AI coding tools: Cursor, GitHub Copilot, Devin, and Claude Code. They also examined 18 policy documents from seven providers. Two coders achieved inter-rater reliability of Cohen's κ 0.81 and κ 0.75 — strong agreement by qualitative standards.[^1]

The research question was whether the governance artefacts of these tools — the files, logs, approvals, and records that a workflow produces — create a traceable link between an AI-generated output and the human accountable for it. The answer is consistent across all four tools: they do not.

The gap is structural, not incidental.

---

## Two layers that do not talk to each other

Lloyd et al. identify two governance layers in AI coding tool deployments, each with a distinct function, and observe that they operate in isolation from each other.

**Layer 1 — Platform controls.** These are the technical mechanisms that define what an agent may do: AGENTS.md directives, approval policies, tool permission configurations, sandbox restrictions, and hook definitions. Platform controls constrain agent behaviour. They are machine-readable, enforceable, and — in well-configured deployments — comprehensive.

**Layer 2 — Provider terms.** These are the contractual and regulatory frameworks that assign responsibility for outputs: terms of service, acceptable use policies, data processing agreements, and enterprise contracts. Provider terms assign accountability to humans. They specify that the user is responsible for reviewing, verifying, and taking responsibility for AI-generated content before it is deployed or shipped.

The structural problem: Layer 1 defines what the agent may do. Layer 2 specifies what the human must verify. Nothing connects them. No artefact records that the verification required by Layer 2 has taken place. No governance record shows which outputs were reviewed, by whom, at what stage, and against what criteria. The platform controls and the provider terms exist in parallel but non-intersecting governance spaces.

---

## The Cursor case study

Lloyd et al.'s most detailed case study examines Cursor, though they note the pattern holds across all four tools examined.

Cursor's terms of service place explicit responsibility on users to verify AI-generated outputs before use. The language is standard for the industry: the user is responsible for reviewing content, ensuring it is appropriate, and bearing responsibility for any outputs that are deployed or acted upon.

Cursor's product records a different event. When a developer clicks "Accept" on a code suggestion, the product records an approval action. The governance artefact — if such a term can even be applied — is the accepted diff in the git history.

The gap: the terms require verification. The product records evaluation and acceptance. These are not the same event. Verification implies that the developer checked the output against an external standard — correctness, security, compliance, architectural fit. Evaluation-and-acceptance records that the developer clicked a button. The two events may coincide, but nothing in the governance artefacts of the tool records which one occurred, or whether the developer had the context to perform verification even if they intended to.

Under EU AI Act Article 9 obligations for high-risk AI systems, or SOX requirements for auditability of financially material systems, or DORA obligations for operational resilience in financial services, this gap is not a minor procedural issue. It is the absence of evidence that the human-in-the-loop requirement has been discharged.

---

## Merge-stage inconsistency across tools

Beyond the Cursor case, Lloyd et al. document significant inconsistency across the four tools in how accountability is assigned at the merge stage — the point where AI-generated code enters a shared repository.

Two of the four tools mandate human approval before merge for any AI-generated content above a defined risk threshold. The approval is recorded in the platform's audit log and is traceable to a named individual.

One tool permits agent approval below defined risk thresholds. The merge is performed by the agent acting autonomously; no human approval is recorded. The governance artefact records the agent's action, not a human decision.

One tool provides advisory guidance that human review is recommended before merge, but does not enforce it at the platform level. Whether review occurred is not recorded.

The practical consequence for enterprise compliance teams: two organisations using different tools from this set operate under fundamentally different governance structures for AI-generated code entering production — without being explicitly told so in any product documentation.

---

## Why this matters now

Three converging pressures are making this gap urgent:

**EU AI Act enforcement.** High-risk AI system deployments under the EU AI Act require technical documentation, human oversight mechanisms, and — crucially — evidence that human oversight has been exercised. AI coding tools used in high-risk contexts (healthcare software, financial systems, critical infrastructure) may qualify as high-risk AI systems. The governance artefact gap means that the required evidence of human oversight does not currently exist in any of the four tools examined.

**SOX and DORA audit requirements.** Organisations subject to Sarbanes-Oxley or the Digital Operational Resilience Act face audit obligations that require traceable human accountability for material system changes. AI-generated code changes in scope for these frameworks. The absence of a governance artefact linking the change to a named human's verified review creates an audit gap.

**Insurance and liability.** Cyber insurance and professional liability policies are beginning to ask specific questions about AI tool use in development. Insurers asking "can you demonstrate that AI-generated code was reviewed before deployment" are asking a question that current governance artefacts cannot answer.

---

## What a complete governance artefact would contain

Lloyd et al. propose that a governance artefact capable of answering "who is accountable for this AI-generated output" should record five elements:

1. **The AI-generated output itself** — the code, suggestion, or artefact produced
2. **The platform controls in effect at generation time** — what the agent was and was not permitted to do
3. **The review event** — that a human reviewed the output, the identity of the reviewer, and the timestamp
4. **The verification criterion** — what standard the reviewer applied (correctness, security, architectural fit)
5. **The acceptance decision** — that the human chose to accept the output and take responsibility for it

Currently, git history satisfies element 1. AGENTS.md partially satisfies element 2. Elements 3, 4, and 5 are absent from the governance infrastructure of every tool examined.

---

## Building the missing layer in Codex CLI

Codex CLI does not provide element 3, 4, or 5 natively. They can be constructed from existing Codex infrastructure.

### PostToolUse logging for element 2

A PostToolUse hook that records platform controls at generation time writes the governance context that review will need:

```bash
#!/bin/bash
# Record platform controls in effect at each tool use
LOG_FILE="${CODEX_REPO_ROOT:-$(pwd)}/.codex/governance-log.jsonl"
mkdir -p "$(dirname "$LOG_FILE")"
cat >> "$LOG_FILE" << EOF
{
  "ts": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "session": "${CODEX_SESSION_ID}",
  "tool": "${CODEX_TOOL_NAME}",
  "path": "${CODEX_TOOL_PATH:-}",
  "approval_policy": "${CODEX_APPROVAL_POLICY:-unset}",
  "agents_md_hash": "$(sha256sum ${CODEX_REPO_ROOT:-$(pwd)}/AGENTS.md 2>/dev/null | awk '{print $1}' || echo 'absent')"
}
EOF
```

This creates a machine-readable record of what constraints were in place when each tool call was made — the closest available approximation to element 2.

### TASK-REVIEW.md for elements 3, 4, and 5

Adding a required task completion artefact to AGENTS.md creates the scaffold for human review documentation:

```markdown
## Task completion artefacts

Every task must produce a TASK-REVIEW.md in the repository root before the task is considered complete. This file must contain:

- Summary of changes made
- List of files modified
- Automated checks that passed (test suite, lint, security scan)
- Any departures from AGENTS.md constraints and the reason for each
- Open questions requiring human judgment before merge

The TASK-REVIEW.md is the primary document for human review. It is the reviewer's responsibility to:
1. Verify the automated checks listed actually passed
2. Confirm the departures listed are acceptable
3. Resolve the open questions before merge
4. Record their decision in the PR description
```

The PR description then carries elements 3 and 4 — the review event and the verification criterion. The merge itself carries element 5. The combination — governance-log.jsonl + TASK-REVIEW.md + PR description + merge record — provides a traceable chain from AI generation to human accountability.

### Enforcement via branch protection

For organisations with formal compliance requirements, GitHub branch protection rules that require a specific PR template and a named approver close the enforcement gap that Lloyd et al. identify. The platform control (branch protection) enforces that a human approval event occurs and records the identity of the approver. Combined with the TASK-REVIEW.md artefact, this provides the evidence trail that current AI coding tools do not produce natively.

---

## The broader implications

Lloyd et al. close their paper with an observation that is worth quoting: "The governance infrastructure of current AI coding tools was designed for a world in which AI suggestions are reviewed by a human who has full context of the codebase and the task. As deployment scales and agents operate autonomously across longer horizons, that assumption is increasingly violated. The tools have not updated their governance models to reflect the change in how they are used."[^1]

The governance artefact gap is not a temporary oversight. It reflects a design era in which the primary concern was productivity and the secondary concern was correctness — and in which accountability was assumed to be a human constant rather than something that needed to be engineered into the workflow.

As AI coding tools move from suggestion-level assistance to autonomous task completion, the accountability model needs to shift accordingly. The five-element governance artefact that Lloyd et al. propose is not a regulatory burden — it is the minimum viable audit trail for organisations that need to demonstrate that human judgment was exercised before AI-generated code entered production.

The tools do not currently provide it. Engineers and compliance teams need to build it themselves, using the infrastructure that tools like Codex CLI make available: AGENTS.md controls, PostToolUse logging, structured PR templates, and branch protection enforcement. The pieces exist. The governance model that assembles them does not yet come in the box.

---

## References

[^1]: Lloyd, A., Hernandez, C., Park, S., and Nakamura, T. "Where Accountability Lives: Mapping Human Responsibility to Workflow Artifacts in Agentic Software Development." arXiv:2608.15678, August 2026. <https://arxiv.org/abs/2608.15678>
