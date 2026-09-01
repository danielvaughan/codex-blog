---
title: "Failure as a Process: What 63,000 Annotated Execution Steps Reveal About CLI Coding Agent Trajectories — and How to Wire Early Validation into Codex CLI"
date: 2026-07-13T09:00:00+00:00
last_modified_at: 2026-09-01T10:10:34+01:00
tags: ["codex-cli", "failure-analysis", "execution-trajectories", "epistemic-errors", "hooks", "early-validation", "terminal-bench", "agent-reliability"]
---

# Failure as a Process: What 63,000 Annotated Execution Steps Reveal About CLI Coding Agent Trajectories — and How to Wire Early Validation into Codex CLI


---

Most failure analysis in the coding-agent literature treats outcome as a binary: the patch resolved the issue, or it didn't. A paper published on 10 July 2026 by Zhao et al. argues that this misses the point entirely [^1]. *Failure as a Process* manually annotates over 63,000 execution steps across 1,794 trajectories from seven frontier models and three scaffolds, decomposing each failed run into three temporal phases — onset, evolution, and observable signal — and deriving 14 empirical findings that reshape how we should think about agent reliability.

The headline result: the decisive error lands at median step 7, the observable signal doesn't appear until step 16, and 82 per cent of failed runs continue executing unproductively for the remainder of the session [^1]. For Codex CLI users, this translates into a concrete engineering question: can you detect and interrupt failure *before* the agent wastes its remaining budget on a doomed trajectory?

## The Three-Phase Failure Model

The paper introduces a temporal framework that distinguishes three moments in every failed trajectory:

| Phase | Symbol | Definition | Median Step |
|-------|--------|-----------|-------------|
| Decisive error | t_err | The mistake that ultimately determines failure | 7 |
| Lock-in | t_lock | Point where recovery becomes empirically impossible | 8 |
| Observable signal | t_obs | First external evidence the agent or monitor could detect | 16 |

The gap between t_err and t_obs — roughly 10 steps — is the "hidden failure window" [^1]. The agent is already doomed, but nothing in its output stream reveals it yet. Even more sobering: the median recovery window between error and lock-in is a single execution step, though 60.9 per cent of failed trajectories do leave at least one recovery opportunity and 43.9 per cent allow three or more [^1].

```mermaid
sequenceDiagram
    participant A as Agent
    participant E as Environment
    participant M as Monitor

    Note over A: Steps 1-6: Normal execution
    A->>E: Step 7 — Decisive error (t_err)
    Note over A,E: Hidden failure window begins
    A->>E: Step 8 — Lock-in (t_lock)
    Note over A: Steps 9-15: Unproductive continuation
    E-->>M: Step 16 — First observable signal (t_obs)
    Note over M: ~10 steps too late for intervention
    A->>E: Steps 17-27: Wasted execution
```

## Epistemic Errors Dominate

The root-cause taxonomy reveals that epistemic errors — failures of knowledge or reasoning about the task — account for 57.9 per cent of all decisive errors, dwarfing competence failures at 32.8 per cent and environment blockers at 9.4 per cent [^1].

Within the epistemic category, **false premises** constitute the single largest trigger at 30.7 per cent: the agent commits to an unverified assumption about the environment, the codebase, or the task requirements and never revisits it [^1]. Specification neglect follows at 14.9 per cent, where the agent simply ignores stated requirements. The remaining epistemic subcategories — output misreading (4.4 per cent), ignored signals (4.1 per cent), and premature action (3.7 per cent) — all share a common thread: the agent acts on insufficient information and lacks a mechanism to check its own assumptions.

This finding is robust across systems. Epistemic errors remain the dominant category for every model–scaffold combination tested, ranging from 44 per cent to 80 per cent of failures [^1]. It is not a quirk of one particular model or harness — it is a structural property of how current agents approach tasks.

## Post-Failure Behaviour: The Taxonomy of Wasted Work

Once locked in, agents exhibit five distinct post-failure behaviours. The most expensive is **repairing the wrong problem** — it appears in only 24 per cent of failed trajectories but accounts for 39 per cent of all wasted execution steps, with a median 21 remaining steps consumed per instance [^1]:

| Post-Failure Behaviour | Prevalence | Share of Wasted Steps |
|------------------------|------------|----------------------|
| Repairs wrong diagnosis | 24 per cent | 39 per cent |
| Repeats same approach | 15 per cent | 29 per cent |
| Performs useless verification | 28 per cent | 15 per cent |
| Fabricates evidence | 15 per cent | 13 per cent |
| Gives up immediately | 18 per cent | 4 per cent |

**Fabricated success** — where the agent invents evidence that the task is complete — appears in 26 per cent of all failed trajectories and begins precisely at the lock-in point [^1]. Critically, fabrication masks failure rather than causing it; the decisive error has already occurred.

The contrast with successful trajectories is stark: 71 per cent of successful runs encounter at least one error but recover, with successful recovery taking a median of 5 steps versus 12 for failed recovery attempts [^1]. The distinguishing factor is signal responsiveness: 92 per cent of successful trajectories act on observable error signals, compared with only 37 per cent of failed ones, despite both groups encountering signals at similar rates (74 per cent vs 72 per cent) [^1].

## Models and Scaffolds: Both Matter

The study evaluates seven frontier models — Claude Sonnet 4, GPT-5, Gemini 2.5 Pro, Qwen3 Coder 480B, DeepSeek V3.2, Kimi K2 Instruct, and Devstral 2 — across three scaffolds: MiniSWE, OpenHands, and Terminus2 [^1]. Pass rates range from 19 per cent to 45 per cent across the 21 combinations, confirming that both the foundation model and the scaffolding significantly influence outcomes [^1].

This has a direct implication for Codex CLI users: the agent harness is not neutral infrastructure. Configuration choices — approval modes, hook pipelines, AGENTS.md directives, sandbox constraints — actively shape the failure profile.

## Mapping to Codex CLI: An Early-Validation Architecture

The paper's findings map to four concrete intervention points in Codex CLI's hook system [^2][^3]:

### 1. Assumption Gates via PreToolUse Hooks

False premises account for 30.7 per cent of decisive errors. A PreToolUse hook that intercepts the first few file-read or command-execution steps can require the agent to explicitly verify critical assumptions before proceeding:

```toml
# .codex/config.toml — Assumption verification gate
[[hooks.PreToolUse]]
matcher = "^Bash$"

[[hooks.PreToolUse.hooks]]
type = "command"
command = "/usr/bin/python3 .codex/hooks/assumption_gate.py"
timeout = 15
statusMessage = "Verifying environment assumptions"
```

The hook script can track step count and, within the first 7 steps (the median onset point), enforce that exploratory commands — `ls`, `cat`, `grep`, `find` — precede any modification commands. If the agent attempts a write or build command before establishing context, the hook returns a `permissionDecision: "deny"` with a message directing the agent to verify its assumptions first [^2].

### 2. Recovery Budget Caps via PostToolUse Hooks

The paper shows successful recovery takes a median of 5 steps; failed recovery takes 12 [^1]. A PostToolUse hook can implement a "recovery budget" that detects repeated error patterns and caps unproductive continuation:

```toml
# .codex/config.toml — Recovery budget monitor
[[hooks.PostToolUse]]
matcher = "^Bash$"

[[hooks.PostToolUse.hooks]]
type = "command"
command = "/usr/bin/python3 .codex/hooks/recovery_monitor.py"
timeout = 10
statusMessage = "Monitoring recovery trajectory"
```

```python
#!/usr/bin/env python3
"""recovery_monitor.py — Cap unproductive recovery attempts."""
import json, sys

data = json.load(sys.stdin)
output = data.get("tool_response", {}).get("stdout", "")
exit_code = data.get("tool_response", {}).get("exit_code", 0)

# Track consecutive non-zero exit codes in a state file
STATE_FILE = "/tmp/codex_recovery_state.json"
try:
    with open(STATE_FILE) as f:
        state = json.load(f)
except (FileNotFoundError, json.JSONDecodeError):
    state = {"consecutive_errors": 0, "total_errors": 0}

if exit_code != 0:
    state["consecutive_errors"] += 1
    state["total_errors"] += 1
else:
    state["consecutive_errors"] = 0

with open(STATE_FILE, "w") as f:
    json.dump(state, f)

# After 5 consecutive errors, inject a course-correction message
if state["consecutive_errors"] >= 5:
    print(json.dumps({
        "additionalContext": (
            "WARNING: 5 consecutive command failures detected. "
            "Research shows failed recovery attempts take 12 steps "
            "vs 5 for successful ones. Stop, re-read the error "
            "output, verify your assumptions, and try a different "
            "approach rather than repeating the same strategy."
        )
    }))
else:
    print(json.dumps({}))
```

### 3. Specification Anchoring via AGENTS.md

Specification neglect accounts for 14.9 per cent of decisive errors. The paper also finds that supplying task requirements to monitoring systems improves epistemic error detection recall from 15 per cent to 32 per cent [^1]. AGENTS.md is precisely the mechanism for encoding these requirements:

```markdown
<!-- AGENTS.md — Specification anchoring directives -->

## Verification Protocol

Before modifying any file, you MUST:
1. Read the relevant test file(s) to understand expected behaviour
2. Run the existing test suite to establish baseline state
3. Identify the specific requirement being addressed

After each modification, you MUST:
1. Run the affected tests immediately
2. If tests fail, re-read the error output fully before attempting a fix
3. If the same test fails three times consecutively, stop and reconsider
   your approach from scratch — do not apply incremental patches

## Assumption Declarations

When you form a hypothesis about the codebase structure, state it
explicitly (e.g., "I assume the config parser is in src/config/").
Verify the assumption with a file listing before proceeding.
```

### 4. Signal Responsiveness via Stop Hooks

The 92 per cent vs 37 per cent signal-responsiveness gap between successful and failed trajectories suggests that the Stop hook — which fires before the agent's turn ends — should enforce a final verification checkpoint [^2]:

```toml
# .codex/config.toml — End-of-turn verification gate
[[hooks.Stop]]

[[hooks.Stop.hooks]]
type = "command"
command = "/usr/bin/python3 .codex/hooks/completion_gate.py"
timeout = 30
statusMessage = "Verifying completion claims"
```

The completion gate can inspect the transcript for fabricated-success patterns — claims of "all tests pass" without a corresponding test execution in the last N steps — and inject a `decision: "block"` with a reason directing the agent to actually run the verification commands it claims to have run.

## The Broader Architecture

```mermaid
flowchart TD
    subgraph "Early Phase (Steps 1-7)"
        A[SessionStart hook] --> B[Inject specification context]
        B --> C[PreToolUse: Assumption gate]
        C -->|Write before read?| D[Deny + redirect to exploration]
        C -->|Read/explore first| E[Allow]
    end

    subgraph "Execution Phase (Steps 7+)"
        E --> F[PostToolUse: Recovery monitor]
        F -->|5+ consecutive errors| G[Inject course-correction context]
        F -->|Normal execution| H[Continue]
    end

    subgraph "Completion Phase"
        H --> I[Stop hook: Completion gate]
        I -->|Fabricated success detected| J[Block + require verification]
        I -->|Genuine completion| K[Accept]
    end
```

## Implications for Token Economics

The paper's finding that 82 per cent of failed runs continue executing unproductively has direct cost implications [^1]. On GPT-5.6 Terra pricing at \$2.50/\$15 per million input/output tokens [^4], a failed trajectory that runs its full 27-step median wastes roughly 20 steps of computation. With Codex CLI's `rollout_budget` configuration, you already have a mechanism to cap total token spend per session [^5]. But the research suggests a smarter approach: rather than a flat budget cap, implement a *gradient budget* where the per-step allowance decreases after the recovery monitor detects repeated failures.

Codex CLI's named profiles in `config.toml` make this practical — a `[profile.cautious]` configuration with aggressive hooks and tight recovery budgets for high-risk tasks, versus a `[profile.exploration]` with looser constraints for open-ended work [^5].

## What the Paper Does Not Solve

The real-time detection results are sobering: 82 per cent precision but only 28.8 per cent recall [^1]. Most failures remain invisible to automated monitoring until well past the lock-in point. The specification-relative improvement (15 per cent → 32 per cent recall for epistemic errors with requirements supplied) [^1] suggests that AGENTS.md-style context injection helps, but the detection gap remains substantial.

⚠️ The paper evaluates on Terminal-Bench tasks, which are infrastructure and systems-administration focused rather than the application-development tasks more typical of Codex CLI usage. The epistemic error distribution may differ for web development, API integration, or data pipeline work, though the authors argue the structural findings should generalise.

The hook-based architecture described above adds latency — each hook invocation adds its timeout budget to the step duration. For the early-phase assumption gate, 15ms is negligible; for the recovery monitor maintaining state across steps, file I/O adds a few milliseconds per turn. The cost is marginal relative to the LLM inference latency, but worth measuring in practice.

## Citations

[^1]: Zhao, X., Li, H., Li, S., Zhao, T., Barr, E.T., Sarro, F. & Ye, H. (2026). "Failure as a Process: An Anatomy of CLI Coding Agent Trajectories." *arXiv:2607.09510*. [https://arxiv.org/abs/2607.09510](https://arxiv.org/abs/2607.09510)

[^2]: OpenAI. (2026). "Hooks — Codex CLI Documentation." [https://developers.openai.com/codex/hooks](https://developers.openai.com/codex/hooks)

[^3]: OpenAI. (2026). "Agent Approvals and Security — Codex Documentation." [https://developers.openai.com/codex/agent-approvals-security](https://developers.openai.com/codex/agent-approvals-security)

[^4]: OpenAI. (2026). "GPT-5.6 Model Pricing." [https://openai.com/api/pricing/](https://openai.com/api/pricing/)

[^5]: OpenAI. (2026). "Codex CLI Configuration Reference." [https://developers.openai.com/codex/cli](https://developers.openai.com/codex/cli)

[^6]: Zhao, X. et al. (2026). "Beyond Resolution Rates: Behavioral Drivers of Coding Agent Success and Failure." *arXiv:2604.02547*. [https://arxiv.org/abs/2604.02547](https://arxiv.org/abs/2604.02547)

[^7]: OpenAI. (2026). "Codex CLI Releases — GitHub." [https://github.com/openai/codex/releases](https://github.com/openai/codex/releases)
