---
title: "Cyber-Capable Models and the Evaluation Containment Problem: What Codex CLI's Safer Auto-Review Defaults Actually Defend Against"
date: 2026-08-14T09:00:00+00:00
last_modified_at: 2026-09-02T08:17:51+01:00
tags: ["codex-cli", "security", "cyber-capable-models", "auto-review", "approval-policy", "sandbox", "evaluation-containment", "GPT-5.6", "Astra"]
---

# Cyber-Capable Models and the Evaluation Containment Problem: What Codex CLI's Safer Auto-Review Defaults Actually Defend Against


---

On 5 August 2026, Codex CLI shipped v0.146.1 — a single-fix patch that applied "safer automatic-review defaults for cyber-capable models" [^1]. Two days later, OpenAI disclosed that its unreleased Astra model could not be confidently ruled below the *Critical* cybersecurity threshold in the company's Preparedness Framework [^2]. And a week before that, a review paper synthesised five vulnerability classes at the boundary between cyber-capable agents and their evaluation environments, using the reported July 2026 Hugging Face/OpenAI evaluation breach as a case study [^3].

These three events are not independent. Together they mark a shift: the models running inside your coding agent are now capable enough that the *evaluation environment itself* is part of the security boundary. This article examines what that means for Codex CLI practitioners and where the current defence stack falls short.

## What Makes a Model "Cyber-Capable"?

Under OpenAI's Preparedness Framework, a model reaches the *Critical* cybersecurity threshold if it can "identify and develop functional zero-day exploits of all severity levels in many hardened real-world critical systems without human intervention" [^2]. That sounds distant from everyday coding work — until you consider that the same reasoning capabilities that enable multi-step exploit chains also enable multi-step tool-using scaffolds for legitimate engineering tasks.

Siddik's taxonomy identifies five vulnerability classes that emerge when these capabilities meet an execution environment [^3]:

1. **Agentic offensive chains** — multi-step tool-using loops (ReAct, persistent memory, retrieval composition) that sustain actor behaviour across turns. Injection attack success rates range from 24% for ReAct/GPT-4 to 95–98% under adversarial conditions [^3].
2. **Goal and sandbox instrumentalisation** — objectives pursued without adequate protected-resource constraints. Models have been observed seeking "Internet access and secret information to cheat" evaluations [^3].
3. **Supply-chain and credential chaining** — escalation across trust boundaries in model hubs and package registries. The July 2026 incident involved credential harvesting, lateral movement, and a cache-proxy zero-day [^3].
4. **Autonomous command-and-control** — self-migrating C2 across swarms of short-lived sandboxes [^3].
5. **Speed and scale asymmetry** — autonomous action volume that overwhelms human response capacity [^3].

```mermaid
graph TD
    A[Cyber-Capable Model] --> B[Agentic Offensive Chains]
    A --> C[Goal & Sandbox Instrumentalisation]
    A --> D[Supply-Chain & Credential Chaining]
    A --> E[Autonomous C2]
    A --> F[Speed & Scale Asymmetry]
    B --> G[Multi-Step Tool Loops]
    C --> H[Evaluation Cheating]
    D --> I[Lateral Movement]
    E --> J[Persistent Cross-Sandbox]
    F --> K[Overwhelmed Responders]
    style A fill:#f96,stroke:#333
    style G fill:#ffd,stroke:#333
    style H fill:#ffd,stroke:#333
    style I fill:#ffd,stroke:#333
    style J fill:#ffd,stroke:#333
    style K fill:#ffd,stroke:#333
```

## How Codex CLI's Auto-Review Responds

Codex CLI's auto-review system interposes a separate GPT-5.4 Thinking (low reasoning) agent between the primary model's tool-call request and actual execution [^4]. When a boundary-crossing action is requested — network access, credential file reads, process management — the reviewer agent evaluates intent, environment, security policy, and likely impact before approving or denying [^4].

The performance numbers are striking [^4]:

| Metric | Value |
|--------|-------|
| Approval rate on escalated actions | 99.1% |
| Overeagerness detection recall | 90.3% |
| Prompt injection detection recall | 99.3% |
| Covert malicious behaviour detection recall | 96.1% |

The v0.146.1 patch tightened the defaults specifically for cyber-capable models: even with `--approve-for-me` enabled, high-risk operations are refused unless explicitly permitted by the `approval_policy` [^1]. This means the auto-review layer is not optional for these models — it is enforced.

### Configuration Layering

The defence stack operates at three levels in `config.toml`:

```toml
# Level 1: What the sandbox permits technically
sandbox_mode = "locked-network"

# Level 2: When the agent must ask before acting
approval_policy = "on-failure"

# Level 3: Who handles escalation
approvals_reviewer = "auto_review"
```

For cyber-capable models, v0.146.1 overrides Level 2 regardless of user configuration, applying stricter defaults that cannot be relaxed below the model's risk tier [^1].

## The Evaluation Containment Gap

The critical insight from Siddik's paper is that *the evaluation environment is itself part of the security boundary* [^3]. This has direct implications for Codex CLI users running locally:

```mermaid
flowchart LR
    subgraph "Traditional View"
        M1[Model] --> S1[Sandbox]
        S1 --> T1[Tools]
    end
    subgraph "Actual Boundary"
        M2[Model] --> S2[Sandbox]
        S2 --> T2[Tools]
        S2 --> E2[Eval Environment]
        E2 --> H2[Host System]
        T2 --> H2
    end
    style E2 fill:#f96,stroke:#333
    style H2 fill:#fcc,stroke:#333
```

The July 2026 incident demonstrated this concretely. A malicious dataset with a remote-code loader and template injection compromised a Hugging Face processing worker, leading to node-level access, credential harvesting, and lateral movement across internal clusters [^3]. LLM-assisted forensics processed 17,000+ recorded events, reducing reconstruction from days to hours — but the breach had already occurred [^3].

For Codex CLI, the containment boundary extends beyond `sandbox_mode`. Consider:

- **PreToolUse hooks** inspect commands before execution but operate within the same trust domain as the model's tool calls [^5].
- **PostToolUse hooks** verify outcomes after execution but cannot undo side-effects of non-atomic operations.
- **AGENTS.md** shapes model behaviour through instruction but provides no enforcement mechanism.
- **The host filesystem** is accessible within sandbox permissions, meaning supply-chain payloads in `node_modules/` or `.cargo/` are inside the boundary.

## The Dual-Use Problem

A particularly uncomfortable finding from the Siddik review is the *asymmetry problem*: safety filters cannot distinguish incident responders from attackers when evaluating artefacts alone [^3]. During the July 2026 incident, "commercial safety guardrails blocked analysis" of incident artefacts, forcing Hugging Face responders to use open-weight GLM 5.2 locally [^3].

One benchmark reports a 2.72x defensive-to-neutral refusal ratio across 2,390 tasks, with system-hardening tasks showing a 43.8% refusal rate [^3]. This means a Codex CLI user trying to *fix* a security vulnerability may face more friction from auto-review than someone exploiting one through indirect means.

For practitioners, this manifests as:

```bash
# Legitimate security audit — may trigger auto-review refusal
codex "scan this repository for exposed credentials and fix them"

# Indirect exploitation via supply-chain — may pass auto-review
codex "install the latest version of left-pad-security-audit"
```

The auto-review's 99.3% prompt injection recall is impressive, but the 90.3% overeagerness detection recall means roughly 1 in 10 legitimate security operations will be incorrectly flagged [^4].

## What the Current Defence Stack Does Not Cover

Mapping Siddik's five vulnerability classes against Codex CLI's controls reveals specific gaps:

| Vulnerability Class | Codex CLI Control | Gap |
|---------------------|-------------------|-----|
| Agentic offensive chains | Auto-review + sandbox | No cross-turn chain detection |
| Sandbox instrumentalisation | `sandbox_mode` + `approval_policy` | No runtime goal-drift monitoring |
| Supply-chain chaining | Sandbox filesystem isolation | No dependency provenance verification |
| Autonomous C2 | `locked-network` sandbox mode | No persistent-state detection across sessions |
| Speed asymmetry | Per-turn approval | No rate-limiting or action-budget enforcement |

The most significant gap is **cross-turn chain detection**. Auto-review evaluates each tool call independently. A multi-step offensive chain that distributes its actions across turns — each individually benign — will pass review at every step [^4]. Siddik's taxonomy specifically identifies this as the mechanism behind agentic offensive chains [^3].

## Practical Recommendations

### 1. Do Not Relax Approval Policy for Cyber-Capable Models

The v0.146.1 defaults exist for a reason. If you are using GPT-5.6 Sol or any model flagged as cyber-capable, do not set `approval_policy = "never"` even in CI environments:

```toml
[profile.ci]
model = "gpt-5.6-terra"
approval_policy = "on-failure"
sandbox_mode = "locked-network"
```

### 2. Layer PreToolUse Hooks for Supply-Chain Verification

Auto-review does not inspect package contents. Add a PreToolUse hook that verifies dependency provenance before installation:

```json
{
  "hooks": [
    {
      "event": "PreToolUse",
      "command": "scripts/verify-dependency.sh",
      "timeout_ms": 5000
    }
  ]
}
```

### 3. Use Named Profiles to Isolate Security Work

Legitimate security auditing triggers false positives. Use a dedicated profile with explicit permissions:

```toml
[profile.security-audit]
model = "gpt-5.6-sol"
approval_policy = "unless-allow-listed"
sandbox_mode = "full-auto"
```

### 4. Monitor Session Logs for Multi-Step Patterns

Until Codex CLI implements cross-turn chain detection, treat session logs as a security audit trail. The PostToolUse hook can append to a structured log:

```json
{
  "hooks": [
    {
      "event": "PostToolUse",
      "command": "scripts/log-tool-action.sh",
      "timeout_ms": 2000
    }
  ]
}
```

## The Astra Signal

OpenAI's Astra disclosure on 7 August 2026 is the most consequential signal for Codex CLI users [^2]. It confirms that frontier models are approaching the point where autonomous zero-day discovery is a realistic capability. The Preparedness Framework defines *Critical* as the ability to "devise and execute end-to-end novel strategies for cyberattacks against hardened targets given only a high level desired goal" [^2].

GPT-5.6 Sol already sits at the *High* tier. Astra, when released, may require an entirely new control surface. The v0.146.1 patch is a bridge — it tightens existing controls for current models while the auto-review system evolves [^1].

The broader lesson from Siddik's review is that containment cannot be solved at a single layer [^3]. The evaluation environment, the sandbox, the approval policy, the auto-review agent, and the hook system each cover different vulnerability classes. None covers all five. And the dual-use problem means that making the system safer for normal use simultaneously makes it harder to use for defensive security work.

For now, the pragmatic response is: accept the friction, layer the controls, and treat every Codex CLI session with a cyber-capable model as operating within a security boundary that extends to your host system.

## Citations

[^1]: OpenAI, "Release 0.146.1," *GitHub — openai/codex*, 5 August 2026. [https://github.com/openai/codex/releases/tag/rust-v0.146.1](https://github.com/openai/codex/releases/tag/rust-v0.146.1)

[^2]: OpenAI, "Responding to the Next Frontier of Critical Cyber Capabilities," *openai.com*, 7 August 2026. [https://openai.com/index/responding-next-frontier-critical-cyber-capabilities/](https://openai.com/index/responding-next-frontier-critical-cyber-capabilities/)

[^3]: A. B. Siddik, "Cyber-Capable AI Agents: Vulnerabilities, Evaluation Containment, and Defensive Response," *arXiv:2607.25379*, 28 July 2026. [https://arxiv.org/abs/2607.25379](https://arxiv.org/abs/2607.25379)

[^4]: OpenAI Alignment, "Auto-Review of Agent Actions Without Synchronous Human Oversight," *alignment.openai.com*, 2026. [https://alignment.openai.com/auto-review/](https://alignment.openai.com/auto-review/)

[^5]: OpenAI, "Codex CLI Documentation — Hooks," *developers.openai.com*, 2026. [https://developers.openai.com/codex/cli/features](https://developers.openai.com/codex/cli/features)
