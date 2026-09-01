---
title: "SkillSec-Eval and the Lifecycle Attack Surface: What 327 Real-World Skills Reveal About Codex CLI's Defence Gaps"
date: 2026-07-25T09:00:00+00:00
last_modified_at: 2026-09-01T20:11:50+01:00
tags: ["codex-cli", "security", "agent-skills", "SkillSec-Eval", "lifecycle-security", "MCP", "supply-chain", "defence-in-depth"]
---

# SkillSec-Eval and the Lifecycle Attack Surface: What 327 Real-World Skills Reveal About Codex CLI's Defence Gaps


---

## The Skill Lifecycle Is the Attack Surface

Most security analysis of coding agents fixates on runtime execution — what happens when an agent runs a tool or writes a file. Badhe and Tiwari's SkillSec-Eval framework, published on 15 July 2026, argues this is dangerously narrow[^1]. Their empirical evaluation of 327 real-world skills from a production repository demonstrates that vulnerabilities distribute across the entire skill lifecycle: from the moment a skill enters a registry, through how it is retrieved and selected, into execution, and through every subsequent update. Runtime guards alone cannot close these gaps.

This matters directly for Codex CLI users. Since v0.117.0, Codex has treated skills as first-class workflow primitives — SKILL.md files stored in `~/.agents/skills/` that encode reusable behaviours[^2]. The Agent Skills open standard now spans Codex, Claude Code, Gemini CLI, and Cursor[^3]. When Snyk's ToxicSkills audit found that 26.1% of skills on ClawHub and skills.sh contained at least one vulnerability[^4], the question shifted from *whether* the skill supply chain is a threat to *where exactly* in the lifecycle existing defences fail.

SkillSec-Eval provides the most granular answer to date.

## The Five Lifecycle Stages

SkillSec-Eval decomposes the skill lifecycle into five sequential stages, each representing a distinct trust boundary[^1]:

```mermaid
flowchart LR
    A["Storage &\nAdmission"] --> B["Semantic\nRetrieval"]
    B --> C["Planner\nSelection"]
    C --> D["Execution\nRuntime"]
    D --> E["Skill\nEvolution"]

    style A fill:#e74c3c,color:#fff
    style B fill:#e67e22,color:#fff
    style C fill:#f1c40f,color:#000
    style D fill:#e74c3c,color:#fff
    style E fill:#e67e22,color:#fff
```

1. **Storage and Admission** — a skill is submitted to a repository and must pass structural and semantic checks before becoming available.
2. **Semantic Retrieval** — when a user invokes a task, the system retrieves candidate skills from the registry based on embedding similarity or keyword matching.
3. **Planner Selection** — the LLM selects which retrieved skill(s) to use, relying on descriptions, metadata, and recommendations.
4. **Execution Runtime** — the agent executes the skill's instructions, invoking tools, reading files, making network calls.
5. **Skill Evolution** — the skill is updated over time, and each update must be re-evaluated against the same threat model.

The key insight: an attacker who cannot compromise a skill at runtime can still compromise it at admission, retrieval, or evolution — stages where most existing defences are absent.

## What the Numbers Show

SkillSec-Eval's empirical evaluation across 327 skills (distributed across 15 capability categories with 20–25 skills per category) produced stark results[^1]:

### Storage and Admission

| Defence | Malicious Admission Rate |
|---|---|
| Rules-only baseline | 52.9% |
| Hybrid (rules + LLM semantic analysis) | 7.9% |

The hybrid admission controller combines deterministic structural verification (schema compliance, cryptographic signatures, payload integrity, provenance, dependency metadata, version lineage) with LLM-based semantic consistency analysis[^1]. The 45-percentage-point improvement is dramatic, but the 20.0% false positive rate creates real developer friction.

### Semantic Retrieval

| Attack | Baseline Success | Defended Success |
|---|---|---|
| Keyword stuffing | 46.59% | 7.83% |
| Sybil publication | 93.20% | 26.59% |
| Average malicious skills in Top-5 | 2.84 | 0.27 |

Sybil attacks — flooding a registry with near-identical malicious skills to dominate retrieval results — remain partially effective even with defences. A 26.59% post-defence success rate means roughly one in four Sybil campaigns still places a malicious skill in the retrieval set[^1].

### Planner Selection

| Attack | Baseline Success | Defended Success |
|---|---|---|
| Fake recommendations | 45.64% | 8.72% |
| Misleading descriptions | 27.27% | 0.00% |
| Permission deception | 11.36% | 5.68% |

Metadata sanitisation — stripping injection patterns and fabricated indicators from skill descriptions before the planner sees them — eliminates misleading-description attacks entirely but leaves permission deception partially viable[^1].

### Execution Runtime

Undefended baseline: **100% attack success**. With dynamic taint tracking and policy enforcement: **23% attack success**. The critical weakness is that LLMs internally paraphrase sensitive data, evading string-based taint tracking. The framework achieved only 66.67% taint propagation accuracy[^1].

### Skill Evolution

Malicious update detection hit 92.5%, but the 37.0% false positive rate means over a third of legitimate updates would be flagged — a rate that erodes developer trust in automated review and encourages blanket approval of flagged changes[^1].

## Mapping to Codex CLI's Defence Stack

How does Codex CLI's current architecture address each lifecycle stage?

### Admission: Partial Coverage

Codex CLI does not operate a centralised skill registry with admission controls. Skills are SKILL.md files that users install locally into `~/.agents/skills/`[^2]. The primary admission control is the `untrusted` project marking, which causes Codex to skip all project-scoped `.codex/` layers[^5]:

```toml
# Codex marks unfamiliar projects as untrusted by default
# Project-local hooks, rules, and config are ignored until trusted
```

For plugins (which bundle skills, MCP servers, and app connectors), the `requirements.toml` file provides fleet-level MCP server allowlists[^6]:

```toml
# requirements.toml — distributed via MDM
[mcp_servers]
allowed_stdio = ["approved-server-1", "approved-server-2"]
```

**Gap:** There is no structural or semantic validation of SKILL.md content at installation time. SkillSec-Eval's hybrid admission controller — which reduced malicious admission from 52.9% to 7.9% — has no equivalent in the current Codex CLI architecture.

### Retrieval: Not Applicable (Yet)

Codex CLI currently loads skills from a local directory rather than performing semantic retrieval from a remote registry. This sidesteps the retrieval attack surface entirely, but only because the skill ecosystem has not yet reached the scale where registry-based discovery is the default. As skill marketplaces mature, this stage will become critical.

### Planner Selection: Tool-Level Gating

When a skill invokes MCP tools, Codex CLI provides granular approval controls:

```toml
[mcp_servers.my_server]
enabled_tools = ["safe_tool_a", "safe_tool_b"]
disabled_tools = ["dangerous_tool_c"]
default_tools_approval_mode = "prompt"

[mcp_servers.my_server.tools.risky_tool]
approval_mode = "approve"
```

The `enabled_tools` and `disabled_tools` lists act as a static allowlist/denylist, reducing the tool surface that a compromised skill can access[^7]. The `approval_policy.granular.skill_approval` key gates whether skill-script approval prompts surface at all[^7].

**Gap:** These controls gate which tools a skill can invoke, not which skill is selected. If a planner selects a malicious skill from `~/.agents/skills/`, the tool-level gating fires too late — the attacker's instructions are already in context.

### Execution Runtime: Strong but Incomplete

This is where Codex CLI's defence stack is deepest:

- **Sandbox isolation**: `workspace-write` mode restricts filesystem access to the active workspace, with `.git/`, `.agents/`, and `.codex/` marked read-only[^5].
- **Network proxy enforcement**: All outbound traffic from sandboxed processes routes through `codex-network-proxy`, with domain-level allowlisting[^6].
- **Auto-review**: The Guardian sub-agent evaluates tool calls against a four-tier risk classification (low, medium, high, critical), with critical actions denied outright[^5].
- **PreToolUse/PostToolUse hooks**: Deterministic scripts that fire before and after every tool invocation, enabling custom security checks[^8].

```toml
# Defence-in-depth: layered runtime controls
approval_policy = "on-request"
approvals_reviewer = "auto_review"

[sandbox_workspace_write]
network_access = true

[features.network_proxy]
enabled = true
domains = { "registry.npmjs.org" = "allow", "*" = "deny" }
```

**Gap:** SkillSec-Eval's finding that LLM paraphrasing defeats string-based taint tracking (66.67% accuracy) applies directly here. If a malicious skill exfiltrates data by instructing the LLM to rephrase sensitive content before writing it to an allowed network destination, neither the sandbox nor the network proxy can detect the semantic transformation. The Guardian auto-review operates on tool-call patterns, not on the semantic content of the agent's reasoning.

### Evolution: No Automated Re-Validation

Codex CLI has no mechanism for automatically re-evaluating skills when they are updated. A skill installed from a trusted source today can be replaced with a malicious version tomorrow, and the `~/.agents/skills/` directory has no integrity verification, version pinning, or change-detection pipeline[^2].

SkillSec-Eval's evolution-stage defence treats every update as a new admission event, achieving 92.5% malicious detection but with a 37.0% false positive rate[^1]. The Codex ecosystem lacks even the baseline.

## A Defence-in-Depth Configuration

Given these gaps, what can a Codex CLI operator configure today to maximise lifecycle coverage?

```toml
# 1. Admission: restrict skill sources
[skills]
# Only enable explicitly reviewed skills
config = [
  { path = "~/.agents/skills/reviewed-skill-1", enabled = true },
  { path = "~/.agents/skills/reviewed-skill-2", enabled = true },
]

# 2. Plugin tool surface: minimise exposure
[plugins.my_plugin.mcp_servers.server_1]
enabled_tools = ["read_data", "query_api"]
default_tools_approval_mode = "approve"

# 3. Runtime: sandbox + network proxy + auto-review
approval_policy = "on-request"
approvals_reviewer = "auto_review"
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
network_access = true

[features.network_proxy]
enabled = true
domains = { "api.internal.company.com" = "allow", "*.npmjs.org" = "allow" }

# 4. Fleet enforcement: requirements.toml
# Distribute via MDM to lock MCP server allowlist organisation-wide
```

For skill evolution monitoring, the gap must currently be filled manually — either by version-controlling the `~/.agents/skills/` directory and reviewing diffs before pulling, or by implementing a PostToolUse hook that checksums skill files on each invocation:

```bash
#!/bin/bash
# PostToolUse hook: verify skill integrity
SKILLS_DIR="$HOME/.agents/skills"
CHECKSUM_FILE="$HOME/.codex/skill-checksums.sha256"

if [ -f "$CHECKSUM_FILE" ]; then
    if ! sha256sum --check --quiet "$CHECKSUM_FILE" 2>/dev/null; then
        echo "SKILL INTEGRITY VIOLATION: skill files modified since last verification" >&2
        exit 1
    fi
fi
```

## What Comes Next

The SkillSec-Eval results reinforce a pattern visible across the broader agent-security literature: the execution-time defence stack (sandboxes, approval policies, network proxies) is necessary but insufficient[^9]. The April 2026 paper by the same community identified seven threat categories across seventeen scenarios[^10], and the Snyk ToxicSkills audit confirmed the threat is not theoretical — 1,467 malicious payloads were found across 3,984 real-world skills[^4].

The structural fix is lifecycle-aware admission: cryptographic provenance, semantic consistency analysis, and continuous re-validation on every update. Until Codex CLI ships a first-party admission controller — or the Agent Skills open standard mandates one — operators must treat every skill as untrusted code and layer their own verification around the installation and update workflow.

The numbers from SkillSec-Eval leave no ambiguity. A rules-only admission baseline lets 52.9% of malicious skills through. Runtime taint tracking catches only 77% of execution-stage attacks. Evolution monitoring generates 37% false positives. Defence-in-depth is not a design philosophy; it is an arithmetic necessity.

---

## Citations

[^1]: Badhe, S. & Tiwari, P. (2026). "Agent Skill Security: Threat Models, Attacks, Defenses, and Evaluation." *arXiv:2607.13987*. [https://arxiv.org/abs/2607.13987](https://arxiv.org/abs/2607.13987)

[^2]: OpenAI. (2026). "Skills." *ChatGPT Learn — Codex Documentation*. [https://learn.chatgpt.com/docs/customization/skills](https://learn.chatgpt.com/docs/customization/skills)

[^3]: OpenAI. (2026). "Codex CLI Plugin System: Bundling Skills, MCP Servers, and App Connectors." *Codex Documentation*. [https://learn.chatgpt.com/docs/plugins](https://learn.chatgpt.com/docs/plugins)

[^4]: Snyk. (2026). "Snyk Finds Prompt Injection in 36%, 1467 Malicious Payloads in a ToxicSkills Study of Agent Skills Supply Chain Compromise." *Snyk Blog*. [https://snyk.io/blog/toxicskills-malicious-ai-agent-skills-clawhub/](https://snyk.io/blog/toxicskills-malicious-ai-agent-skills-clawhub/)

[^5]: OpenAI. (2026). "Agent Approvals & Security." *ChatGPT Learn — Codex Documentation*. [https://learn.chatgpt.com/docs/agent-approvals-security](https://learn.chatgpt.com/docs/agent-approvals-security)

[^6]: OpenAI. (2026). "Advanced Configuration." *ChatGPT Learn — Codex Documentation*. [https://learn.chatgpt.com/docs/config-file/config-advanced](https://learn.chatgpt.com/docs/config-file/config-advanced)

[^7]: OpenAI. (2026). "Configuration Reference." *ChatGPT Learn — Codex Documentation*. [https://learn.chatgpt.com/docs/config-file/config-reference](https://learn.chatgpt.com/docs/config-file/config-reference)

[^8]: OpenAI. (2026). "Hooks." *ChatGPT Learn — Codex Documentation*. [https://learn.chatgpt.com/docs/hooks](https://learn.chatgpt.com/docs/hooks)

[^9]: Badhe, S. & Tiwari, P. (2026). "Towards Secure Agent Skills: Architecture, Threat Taxonomy, and Security Analysis." *arXiv:2604.02837*. [https://arxiv.org/abs/2604.02837](https://arxiv.org/abs/2604.02837)

[^10]: Ibid.
