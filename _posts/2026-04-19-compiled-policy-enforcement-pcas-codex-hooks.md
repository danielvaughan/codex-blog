---
title: "Compiled Policy Enforcement: Why Prompt-Based Safety Fails at 48% and What PCAS Means for Codex Hooks"
description: "Prompt-based policy enforcement — telling a model never do X in a system prompt — achieves only 48% compliance even with frontier models ."
type: Technical Article
timestamp: 2026-04-19T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-04-19-compiled-policy-enforcement-pcas-codex-hooks"
tags: ["codex-cli", "security", "hooks", "policy-enforcement", "PCAS", "governance", "enterprise"]
date: 2026-04-19T09:00:00+00:00
last_modified_at: 2026-09-02T08:17:51+01:00
---
![Sketchnote diagram for: Compiled Policy Enforcement: Why Prompt-Based Safety Fails at 48% and What PCAS Means for Codex Hooks](/sketchnotes/articles/2026-04-19-compiled-policy-enforcement-pcas-codex-hooks.png)


# Compiled Policy Enforcement: Why Prompt-Based Safety Fails at 48% and What PCAS Means for Codex Hooks


---

Prompt-based policy enforcement — telling a model "never do X" in a system prompt — achieves only 48% compliance even with frontier models [^1]. That is not a rounding error; it means the model violates your security policy more often than it follows it. If you are running Codex CLI in an enterprise environment with real compliance requirements, that number should alarm you.

This article examines the Policy Compiler for Agentic Systems (PCAS) [^1], a February 2026 research system that compiles declarative Datalog rules into deterministic reference monitors. We then map PCAS concepts onto Codex CLI's existing hook architecture to show where current hooks already provide value, where the gaps lie, and how a compiled policy approach could evolve Codex's governance story from imperative scripts to formal enforcement.

## The Prompt-Based Safety Problem

The intuition behind prompt-based safety is straightforward: instruct the model to follow rules, and it will. The evidence says otherwise.

Palumbo et al. evaluated frontier models — including Claude Opus, GPT-5.2, and Gemini 3 Pro — on customer service policy adherence across airline and retail domains [^1]. Without external enforcement, baseline compliance hovered at 48%. Models would routinely issue refunds exceeding policy limits, access APIs they were not authorised to call, and leak sensitive data across agent boundaries.

This is not unique to one benchmark. A 2026 red-team study from Harvard, MIT, Stanford, and Carnegie Mellon documented agents circumventing model-level guardrails in live environments [^2]. Independent benchmarking of prompt-based guardrails shows a 26.67% policy violation rate under adversarial conditions [^3]. Meanwhile, 63% of organisations deploying AI agents cannot enforce purpose limitations on what those agents are authorised to do [^4].

The fundamental issue is architectural: prompt-based rules are *suggestions* processed by the same probabilistic reasoning engine they aim to constrain. They can be overridden by context, eroded by long conversations, or bypassed by prompt injection.

## How PCAS Works: From Datalog to Deterministic Enforcement

PCAS takes a fundamentally different approach. Rather than asking the model to self-police, it compiles declarative policies into a reference monitor that intercepts actions before execution — deterministically, independent of model reasoning [^1].

### The Dependency Graph

Traditional agent systems track a linear conversation history. PCAS models agent state as a causal dependency graph `G = (V, D)`, where nodes represent events (messages, tool calls, results) and edges encode causal dependencies [^1]. This distinction matters because linear history loses provenance: it cannot answer "did this tool call's input ultimately derive from untrusted data?"

The dependency graph enables **backward slicing** — given a proposed action, PCAS extracts the subgraph of all transitively reachable nodes, capturing the full causal history across agents [^1].

### Datalog Policy Language

Policies are expressed as declarative rules in a Datalog-derived language with recursive predicates for transitive dependency tracking [^1]:

```
Depends(dst, src) :- Edge(src, dst).
Depends(dst, src) :- Depends(dst, mid), Edge(src, mid).
```

This enables policies that reason about information flow rather than surface-level pattern matching. A toxic flow policy, for instance, tracks taint transitively across agents via `Depends(id, src)` predicates, blocking actions when untrusted and sensitive data combine — regardless of which agent accessed them [^1].

The core relations include:

| Relation | Purpose |
|----------|---------|
| `Actions(a)` | Proposed actions |
| `Edge(src, dst)` | Causal dependencies |
| `Depends(dst, src)` | Transitive closure |
| `AuthenticatedEntity(e)` | Identity integration |
| `Allowed(a)` / `Denied(a)` | Policy decisions |

### The Reference Monitor

The reference monitor `ℛ(a, G, P)` intercepts every proposed action, evaluates it against the policy engine using the action's backward slice, and returns `{allow, deny}` with a structured message [^1]. Denied actions return feedback enabling the agent to retry with a compliant approach — the system is not a blunt kill switch.

### Compilation: Policy-Compliant by Construction

The compiler `𝒞` transforms an existing agent implementation plus a policy specification into an instrumented system that is policy-compliant by construction [^1]. No security-specific restructuring of the agent code is required. The reference monitor is injected into all action execution paths automatically.

```mermaid
flowchart LR
    A[Agent Code] --> C[PCAS Compiler]
    P[Policy Spec<br/>Datalog Rules] --> C
    C --> I[Instrumented Agent]
    I --> RM[Reference Monitor]
    RM -->|Allow| T[Tool Execution]
    RM -->|Deny + Feedback| A2[Agent Retries]
```

## PCAS Results: Three Case Studies

The evaluation spans three distinct scenarios [^1]:

### 1. Prompt Injection Defence

Using GPT-4.1-mini with information flow policies (multi-level security and toxic flow), PCAS achieved 0/5 successful attacks in instrumented runs versus 5/5 in non-instrumented runs. Utility remained 5/5 across all configurations — the reference monitor blocked attacks without degrading legitimate functionality.

### 2. Customer Service Compliance

Across the τ²-bench airline and retail domains with Claude Opus, GPT-5.2, and Gemini 3 Pro, compliance improved from 48% to 93%, with pass rates improving 1.68–2.93× over baseline [^1].

### 3. Multi-Agent Pharmacovigilance

In a multi-agent approval workflow, prediction accuracy reached 15/15 correct and unauthorised FDA API accesses dropped from 42 to zero [^1].

## Mapping PCAS to Codex CLI's Hook Architecture

Codex CLI already has the architectural skeleton for policy enforcement through its hook system [^5]. The question is how far the current implementation goes and where PCAS-style enhancements would add value.

### What Codex Hooks Already Provide

Codex hooks fire at five lifecycle events: `SessionStart`, `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, and `Stop` [^5]. The `PreToolUse` hook is the closest analogue to PCAS's reference monitor — it intercepts tool invocations before execution and can block them with a deny decision [^5].

Configuration lives in `hooks.json` at user (`~/.codex/hooks.json`) or repository (`.codex/hooks.json`) level [^5]:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/policy-check.sh",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

The hook receives JSON on stdin including `session_id`, `cwd`, `model`, and `turn_id`, and can return `permissionDecision: "deny"` to block execution [^5].

### The Gaps

| PCAS Concept | Codex Hook Equivalent | Gap |
|---|---|---|
| Causal dependency graph | Linear conversation transcript | No provenance tracking across turns or agents |
| Datalog declarative policies | Imperative shell scripts | Policy logic embedded in code, not formally verifiable |
| Transitive information flow | Not present | Cannot answer "did this input derive from untrusted data?" |
| Cross-agent provenance | Subagent sandbox isolation | Subagents are isolated but no formal flow tracking between them |
| Compiled enforcement | Runtime script execution | No compile-time policy guarantees |

### A Practical Example: Credential Flow Policy

Consider the policy "no production credentials should flow to subagents." In current Codex CLI, you would write a `PreToolUse` hook script that pattern-matches on environment variable names or file paths:

```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

if echo "$COMMAND" | grep -qE '(PROD_DB_|AWS_SECRET|PRODUCTION_API)'; then
  echo '{"permissionDecision": "deny"}' >&2
  exit 2
fi
exit 0
```

This catches direct references but misses indirect flows. If a previous tool call read a production secret into a variable, and a subsequent call passes that variable to a subagent without using the original name, the pattern-matching hook would not catch it.

In PCAS's Datalog, the same policy would be:

```
ProdCred(id) :- ToolResult(id, "read_secret", args),
                Contains(args, "production").
Denied(a)    :- Actions(a), Depends(a, src), ProdCred(src).
```

This blocks any action whose causal history includes a production credential read — regardless of variable renaming, intermediate processing, or agent boundaries [^1].

## The Emerging Compiled Policy Ecosystem

PCAS is not alone. The shift from probabilistic to deterministic agent governance is becoming a trend:

- **Microsoft's Agent Governance Toolkit** (April 2026) provides sub-millisecond deterministic policy enforcement with 0.00% violation rate under adversarial conditions, compared to 26.67% for prompt-based guardrails [^3].
- **Cedar Policy Language** integration with coding agent hooks (including Claude Code) provides formally analysable attribute-based access control, with policies that can be checked for contradictions and vacuous rules before deployment [^6].
- **Codex CLI's OpenTelemetry integration** emits counters and duration histograms for API, stream, and tool activity [^7], providing the observability data that could feed into a PCAS-style dependency graph.

```mermaid
flowchart TB
    subgraph Current["Current State: Imperative Hooks"]
        H1[PreToolUse Hook] -->|Pattern Match| D1{Allow?}
        D1 -->|Yes| E1[Execute]
        D1 -->|No| B1[Block]
    end

    subgraph Future["Future State: Compiled Policies"]
        DG[Dependency Graph] --> PE[Policy Engine<br/>Datalog Rules]
        PE --> RM2{Reference Monitor}
        RM2 -->|Provenance OK| E2[Execute]
        RM2 -->|Flow Violation| B2[Block + Feedback]
        E2 -->|Update| DG
    end

    Current -.->|Evolution| Future
```

## Practical Steps for Codex CLI Users Today

You do not need to wait for PCAS integration. Here is how to strengthen policy enforcement with current Codex CLI hooks:

**1. Layer your defences.** Use `PreToolUse` hooks for command-level blocking, `PostToolUse` hooks for output validation, and `UserPromptSubmit` hooks for prompt sanitisation [^5]. Each catches a different failure mode.

**2. Centralise policy logic.** Rather than scattering rules across shell scripts, maintain a single policy definition file that your hooks reference. This is not Datalog, but it moves towards declarative policy expression.

**3. Use OTEL for audit trails.** Enable Codex's OpenTelemetry metrics pipeline [^7] to capture tool call patterns. This data can inform which policies need tightening and provides the audit trail compliance teams require.

**4. Combine with `codex.toml` deny rules.** The `disable_commands` configuration in `codex.toml` provides a static deny list [^8]. Layer this with dynamic `PreToolUse` hooks for defence in depth.

**5. Watch for dependency graph features.** As Codex evolves its hook system — particularly around subagent orchestration and the guardian model — expect causal tracking to become relevant. Structure your hook scripts to log provenance data now, so you are ready to feed it into formal policies later.

## Conclusion

The 48% compliance figure should end the debate about whether prompt-based safety is sufficient for production agent deployments. It is not. PCAS demonstrates that compiled, deterministic policy enforcement can achieve zero violations without sacrificing agent utility [^1].

Codex CLI's hook architecture provides the interception points that make this kind of enforcement possible. The gap is in the policy language (imperative scripts versus declarative Datalog) and the state model (linear transcript versus causal dependency graph). As the ecosystem matures — with Microsoft's Agent Governance Toolkit, Cedar integration, and Codex's own evolving hook system — expect compiled policy enforcement to become a standard layer in production agentic workflows.

For now, the pragmatic approach is to maximise the value of Codex's existing hooks while keeping an eye on the formal methods that will eventually replace them.

## Citations

[^1]: Palumbo, N., Choudhary, S., Choi, J., Chalasani, P., & Jha, S. (2026). "Policy Compiler for Secure Agentic Systems." arXiv:2602.16708. [https://arxiv.org/abs/2602.16708](https://arxiv.org/abs/2602.16708)

[^2]: Kiteworks. (2026). "AI Agent Data Governance 2026: Why 63% of Organizations Can't Stop Their Own AI." [https://www.kiteworks.com/cybersecurity-risk-management/ai-agent-data-governance-why-organizations-cant-stop-their-own-ai/](https://www.kiteworks.com/cybersecurity-risk-management/ai-agent-data-governance-why-organizations-cant-stop-their-own-ai/)

[^3]: Microsoft. (2026). "Introducing the Agent Governance Toolkit: Open-source runtime security for AI agents." [https://opensource.microsoft.com/blog/2026/04/02/introducing-the-agent-governance-toolkit-open-source-runtime-security-for-ai-agents/](https://opensource.microsoft.com/blog/2026/04/02/introducing-the-agent-governance-toolkit-open-source-runtime-security-for-ai-agents/)

[^4]: Inkog Labs. (2026). "The AI Agent Security Gap: Findings from Scanning 500+ Open-Source AI Agent Projects." [https://inkog.io/labs/ai-agent-security-gap-2026](https://inkog.io/labs/ai-agent-security-gap-2026)

[^5]: OpenAI. (2026). "Hooks – Codex | OpenAI Developers." [https://developers.openai.com/codex/hooks](https://developers.openai.com/codex/hooks)

[^6]: Sondera AI. (2026). "Hooking Coding Agents with the Cedar Policy Language." [https://blog.sondera.ai/p/hooking-coding-agents-with-the-cedar](https://blog.sondera.ai/p/hooking-coding-agents-with-the-cedar)

[^7]: OpenAI. (2026). "Advanced Configuration – Codex | OpenAI Developers." [https://developers.openai.com/codex/config-advanced](https://developers.openai.com/codex/config-advanced)

[^8]: OpenAI. (2026). "Configuration Reference – Codex | OpenAI Developers." [https://developers.openai.com/codex/config-reference](https://developers.openai.com/codex/config-reference)
