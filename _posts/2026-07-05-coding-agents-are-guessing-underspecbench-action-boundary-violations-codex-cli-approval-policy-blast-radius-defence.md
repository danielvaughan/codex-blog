---
title: "Coding Agents Are Guessing: What UnderSpecBench Reveals About Action-Boundary Violations — and How Codex CLI's Approval Architecture Defends Against Them"
date: 2026-07-05T09:00:00+00:00
last_modified_at: 2026-09-06T04:10:35+01:00
tags: ["codex-cli", "underspecification", "action-boundary", "approval-policy", "sandbox", "hooks", "safety", "DevOps", "UnderSpecBench"]
---

# Coding Agents Are Guessing: What UnderSpecBench Reveals About Action-Boundary Violations — and How Codex CLI's Approval Architecture Defends Against Them


---

When a developer tells a coding agent to "restart the failing service," which service does the agent pick? If three candidates match, does it ask? Or does it guess — and restart all three?

Ji et al.'s *UnderSpecBench* (arXiv:2607.02294, July 2026) answers that question empirically: agents guess[^1]. Across 2,208 prompt variants spanning four DevOps domains, 55.8–67.8% of executed runs violated at least one operational boundary[^1]. The paper exposes a class of failure that completion-centric benchmarks systematically miss — and it maps directly onto the approval, sandbox, and hook architecture that Codex CLI provides today.

## The Problem: Underspecification Makes Agents Act, Not Abstain

UnderSpecBench constructs 69 task families grounded in real CVEs, incident reports, and tool documentation[^1]. Each family is instantiated along three underspecification axes:

- **Intent clarity (S₀–S₃):** from fully explicit instructions to heavily ambiguous ones
- **Target certainty (B₀–B₃):** from uniquely identified resources to multiple candidate matches
- **Blast radius (R₀–R₁):** contained operations versus production-facing ones

The 4×4×2 matrix produces 32 variants per family — 2,208 prompts total — evaluated against five agent×model configurations including Claude Code + Haiku-4.5, Codex + Codex-5.1-mini, and OpenCode with three model backends[^1].

### The Core Finding

Target ambiguity is the dominant driver. As target certainty degrades from B₀ to B₃, Safe Success collapses from 67.9% to 8.6%, and Wrong Target violations climb from 9.6% to 75.1%[^1]. Intent ambiguity matters less: Safe Success falls from 50.9% (S₀) to 29.4% (S₃) — a shallower gradient[^1].

The most troubling finding is **blast-radius blindness**: agents showed nearly identical behaviour whether an operation was sandboxed or production-facing. Action rates were 65.5% at R₀ versus 64.0% at R₁[^1]. Safe Success hovered around 42% at both levels. Agents react to *semantic* underspecification but not to *consequence severity*.

```mermaid
graph LR
    subgraph "UnderSpecBench Axes"
        S["Intent Clarity<br/>S₀→S₃"]
        B["Target Certainty<br/>B₀→B₃"]
        R["Blast Radius<br/>R₀→R₁"]
    end
    B -->|"Strong effect<br/>Safe Success 67.9%→8.6%"| Q["Action Quality"]
    S -->|"Moderate effect<br/>Safe Success 50.9%→29.4%"| Q
    R -->|"Negligible effect<br/>~42% at both levels"| Q
    Q --> V["55.8–67.8% boundary<br/>violation rate"]
```

## OverScope Follows Surface Topology

The paper distinguishes two structural categories of control surface[^1]:

| Surface Type | Example | OverScope Rate | Safe Success |
|---|---|---|---|
| Bounded-object | Work governance, repository state | 14.4–37.6% | 24–35% |
| Shared control-plane | Deployment/traffic, infrastructure | 59.8–77.2% | 12.6–16.6% |

On shared control planes — load balancers, DNS records, deployment pipelines — an underspecified instruction does not merely hit the wrong target; it propagates globally. This is precisely where agent-driven DevOps carries the highest risk, and precisely where agents show the least restraint.

## Agents Rarely Refuse

Explicit refusal rates across all configurations were ≤2.5%[^1]. The more common non-action dispositions were clarification questions (Ask) and analysis without commitment (Defer), but these were heavily model- and scaffold-dependent. Haiku-4.5 asked for clarification in 38–45% of non-action cases; DeepSeek-v4 asked in only 1.7%[^1]. Critically, the same model (Codex-5.1-mini) asked 31.8% of the time under its native Codex harness but only 10.5% under OpenCode[^1] — demonstrating that **the harness, not the model, determines whether hesitation surfaces as a useful question**.

This corroborates Qu et al.'s *OverEager-Gen* finding (arXiv:2605.18583) that on Claude Code, removing the consent declaration alone raises the overeager rate from 0.0% to 17.1%[^2]. The harness is the defence.

## How Codex CLI's Architecture Addresses Each Finding

UnderSpecBench proposes three mitigation layers: user practice, model alignment, and harness/system controls[^1]. Codex CLI implements the harness layer comprehensively.

### 1. Approval Policies as Action Gates

Codex CLI's `approval_policy` directly addresses the action-propensity problem[^3]. Three built-in modes map to escalating trust:

```toml
# config.toml — graduated trust
approval_policy = "untrusted"      # everything requires approval
# approval_policy = "on-request"   # sandbox escalations need approval
# approval_policy = "never"        # fully autonomous (CI/batch only)
```

The `on-request` default means any operation that escapes the sandbox boundary — network access, writes outside the workspace, or elevated commands — requires explicit human confirmation[^3]. This is the "confirmation for irreversible operations" that UnderSpecBench recommends at the harness layer.

For finer control, granular policies let teams selectively gate specific approval categories[^3]:

```toml
[approval_policy]
granular = { sandbox_approval = true, rules = true, mcp_elicitations = true, request_permissions = false }
```

### 2. Sandbox Modes as Blast-Radius Containment

UnderSpecBench's most alarming result — blast-radius blindness — is precisely what Codex CLI's sandbox layer addresses. The `workspace-write` default confines the agent to the current working directory with network disabled[^4]:

```toml
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
network_access = false  # default: no network
```

Even within writable mode, `.git`, `.agents/`, and `.codex/` directories remain read-only[^4]. This converts a shared-control-plane surface (where UnderSpecBench observed 59.8–77.2% OverScope) into a bounded-object surface (14.4–37.6%) by architectural constraint rather than model discretion.

For production-facing operations that genuinely require network access, domain allow-lists provide graduated exposure[^4]:

```toml
[features.network_proxy]
enabled = true
domains = { "api.internal.example.com" = "allow", "*" = "deny" }
```

### 3. PreToolUse Hooks as Deterministic Boundary Enforcement

Where approval policies rely on the agent requesting permission, PreToolUse hooks enforce boundaries deterministically before execution[^5]. A hook that intercepts destructive commands does not depend on the model's judgement about consequence severity:

```bash
#!/usr/bin/env bash
# .codex/hooks/block-destructive.sh
# PreToolUse hook: exit 2 blocks the action

INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

# Block production-facing operations
if echo "$COMMAND" | grep -qE '(kubectl delete|terraform destroy|docker rm -f)'; then
  echo "Blocked: destructive infrastructure command requires manual execution" >&2
  exit 2
fi

exit 0
```

Exit code 2 blocks the action and feeds the reason back to the agent as context, so the model can adjust its approach[^5]. This addresses UnderSpecBench's finding that agents do not self-moderate on consequence severity — the hook externalises that judgement into deterministic policy.

Enterprise teams can enforce managed hooks via `requirements.toml` with `allow_managed_hooks_only = true`, preventing individual developers from bypassing organisational safety floors[^5][^6].

### 4. AGENTS.md as Clarification Affordance

UnderSpecBench shows that whether an agent asks for clarification depends on the harness exposing it as a first-class affordance[^1]. In Codex CLI, `AGENTS.md` is the primary mechanism for encoding when the agent should pause and ask:

```markdown
## Operational Boundaries

- Before modifying any production configuration, ASK the user which environment is targeted
- If multiple services match a description, LIST all candidates and ask for confirmation
- Never run `terraform apply` without explicit user approval of the plan output
```

These instructions load into context automatically and cascade through the directory hierarchy — repository root, subdirectories, and home directory[^7]. Combined with the `untrusted` approval policy, they convert the agent's default "guess and act" behaviour into "identify ambiguity and ask."

### 5. Permission Profiles for Surface-Specific Trust

UnderSpecBench's surface-topology finding — that OverScope rates are structurally determined by whether a resource is bounded or shared — suggests different trust levels for different operational contexts. Codex CLI's named permission profiles (since v0.129) encode this directly[^3]:

```toml
# .codex/profiles/infra-readonly.toml
sandbox_mode = "read-only"
approval_policy = "untrusted"
# Used for infrastructure inspection tasks

# .codex/profiles/local-dev.toml
sandbox_mode = "workspace-write"
approval_policy = "on-request"
# Used for local development work
```

Developers switch profiles with `--profile infra-readonly` when working on shared control planes, matching the trust level to the surface topology that UnderSpecBench identifies as the structural driver of OverScope[^3].

## Putting It Together: A Four-Layer Defence

```mermaid
graph TB
    subgraph "UnderSpecBench Failure"
        U["Underspecified<br/>instruction"]
    end
    subgraph "Codex CLI Defence Layers"
        A["AGENTS.md<br/>Clarification rules"]
        P["approval_policy<br/>Action gates"]
        H["PreToolUse hooks<br/>Deterministic blocks"]
        S["Sandbox<br/>Blast-radius containment"]
    end
    U --> A
    A -->|"Agent asks<br/>for clarification"| SAFE["Safe outcome"]
    A -->|"Agent proceeds"| P
    P -->|"Approval required"| SAFE
    P -->|"Auto-approved"| H
    H -->|"Exit 2: blocked"| SAFE
    H -->|"Exit 0: allowed"| S
    S -->|"Confined to workspace"| CONTAINED["Contained<br/>boundary violation"]
    S -->|"Full access"| RISK["⚠️ Unmitigated risk"]
```

Each layer catches what the previous one missed. `AGENTS.md` reduces ambiguity at the semantic level. Approval policies gate actions that escape the sandbox. Hooks enforce hard boundaries regardless of the agent's assessment of consequence. And the sandbox ensures that even a boundary violation remains contained to the working directory.

## Practical Recommendations

1. **Default to `on-request`, not `never`**: UnderSpecBench shows that agents act in 58.5–72.3% of runs even at maximum ambiguity. The `on-request` policy ensures that at least sandbox-escaping actions require human confirmation.

2. **Write target-specification rules in AGENTS.md**: The paper's strongest finding is that target ambiguity (B axis) drives the sharpest quality degradation. Encode explicit "if ambiguous, ask" rules for your project's high-risk targets.

3. **Use PreToolUse hooks for shared control planes**: For deployment, DNS, load balancing, and infrastructure operations — where UnderSpecBench observed 59.8–77.2% OverScope — add deterministic hooks that block or require confirmation regardless of prompt clarity.

4. **Switch permission profiles by context**: Use `read-only` + `untrusted` for production inspection; `workspace-write` + `on-request` for local development. Match trust to surface topology.

5. **Test your boundaries with `codex sandbox`**: Validate that your sandbox configuration actually prevents the operations you intend to block by running `codex sandbox macos --permissions-profile <name> <command>`[^4].

## Citations

[^1]: Ji, Z., Zhang, Z., Xu, C., Tian, Y., Li, Z., Gao, Y., Wang, S. & Cheung, S.-C. (2026). "Coding Agents Are Guessing: Measuring Action-Boundary Violations in Underspecified DevOps Instructions." *arXiv:2607.02294*. [https://arxiv.org/abs/2607.02294](https://arxiv.org/abs/2607.02294)

[^2]: Qu, Y., Zhang, Y., Zhang, Y., Deng, G., Li, Y., Zhang, L.Y. & Liu, Y. (2026). "Overeager Coding Agents: Measuring Out-of-Scope Actions on Benign Tasks." *arXiv:2605.18583*. [https://arxiv.org/abs/2605.18583](https://arxiv.org/abs/2605.18583)

[^3]: OpenAI. (2026). "Agent approvals & security — Codex." *OpenAI Developers*. [https://developers.openai.com/codex/agent-approvals-security](https://developers.openai.com/codex/agent-approvals-security)

[^4]: OpenAI. (2026). "Sandbox — Codex." *OpenAI Developers*. [https://developers.openai.com/codex/concepts/sandboxing](https://developers.openai.com/codex/concepts/sandboxing)

[^5]: OpenAI. (2026). "Hooks — Codex." *OpenAI Developers*. [https://developers.openai.com/codex/hooks](https://developers.openai.com/codex/hooks)

[^6]: OpenAI. (2026). "Configuration Reference — Codex." *OpenAI Developers*. [https://developers.openai.com/codex/config-reference](https://developers.openai.com/codex/config-reference)

[^7]: OpenAI. (2026). "Custom instructions with AGENTS.md — Codex." *OpenAI Developers*. [https://developers.openai.com/codex/guides/agents-md](https://developers.openai.com/codex/guides/agents-md)
