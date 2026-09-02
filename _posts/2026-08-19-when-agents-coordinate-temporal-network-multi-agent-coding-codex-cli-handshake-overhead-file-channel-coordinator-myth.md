---
title: "When Agents Coordinate: What 1,902 Temporal-Network Runs Reveal About Your Codex CLI Multi-Agent Costs"
date: 2026-08-19T09:00:00+00:00
last_modified_at: 2026-09-02T03:11:17+01:00
tags: ["codex-cli", "multi-agent", "coordination", "temporal-network", "handshake-overhead", "file-channel", "orchestration", "token-cost"]
---

# When Agents Coordinate: What 1,902 Temporal-Network Runs Reveal About Your Codex CLI Multi-Agent Costs


---

Running multiple Codex CLI instances in parallel — a pattern the community calls "agentmaxxing" — is now mainstream for senior teams [^1]. But the coordination cost of that parallelism has remained largely unmeasured. A new study from University College London changes that: Destefanis and Aste instrument 1,902 multi-agent coding runs as temporal networks and find that messaging costs scale quadratically with team size, that files can replace nearly half the message tokens on the right task, and that naming a coordinator agent does nothing structurally useful [^2].

This article maps their findings to Codex CLI v0.147.0's MultiAgentV2 orchestration layer, session fork, and Agent Plugins, showing where the current architecture already optimises coordination and where it leaves cost on the table.

## The Instrument: Runs as Temporal Networks

Most multi-agent evaluations report two numbers: task pass rate and total cost. Destefanis and Aste argue these mask the *shape* of coordination. Their instrument represents each run as a temporal network: agents and files are nodes; messages, file writes, and file reads are timestamped directed edges carrying a token cost [^2].

This lets them measure density, clustering coefficient, and degree distribution at each point in the run — and compare those metrics across team sizes (2, 4, 8, 16 agents), task types (distributed specification-reconciliation vs chained pipeline), and file policies (forbidden, allowed, mandatory).

```mermaid
graph LR
    subgraph "Temporal Network Model"
        A1[Agent 1] -->|message 42 tokens| A2[Agent 2]
        A2 -->|write 18 tokens| F1[shared_spec.md]
        A3[Agent 3] -->|read 0 tokens| F1
        A1 -->|message 55 tokens| A3
        A2 -->|message 31 tokens| A3
    end
    subgraph "Metrics Extracted"
        D[Density]
        C[Clustering]
        DG[Degree Distribution]
        TC[Token Cost per Edge]
    end
```

## Finding 1: Quadratic Messaging Is Mostly a Handshake

Direct messaging grows with a scaling exponent of 1.92 on chained tasks — quadratic within error [^2]. On the distributed task, message counts rise from 6.1 at two agents to 28.5 at four and 71.3 at eight. That looks alarming. But the per-pair rate tells the opposite story: it drops from approximately three messages per ordered pair at two agents to 1.27 at eight [^2].

The explanation: 90% of sender-recipient pairs are established by normalised time τ ≈ 0.2 to 0.6. Agents front-load an all-to-all introduction round, then settle into a handful of stable channels for the remainder of the run. At 16 agents, messaging actually plateaus at 46.8 messages — statistically flat (slope = 0.00, 95% CI [−0.34, 0.34]) [^2].

### What This Means for Codex CLI

Codex CLI's MultiAgentV2 system, introduced in v0.128.0, spawns subagents with `fork_turns: "all"` to share the orchestrator's analysis context [^3]. This design inadvertently addresses the handshake problem: by forking from a shared context, subagents skip the introduction round entirely. The orchestrator has already established the project state; forked agents inherit it.

But for workflows using `codex exec` in separate worktrees — the classic agentmaxxing pattern — there is no shared context. Each agent rediscovers the codebase independently. The temporal network data suggests this costs roughly τ = 0.2 to 0.6 of each session's message budget on redundant orientation.

```toml
# config.toml — fork-based orchestration avoids handshake overhead
[multi_agent]
max_threads = 8
fork_turns = "all"          # share orchestrator context
wait_timeout_seconds = 300

[model]
model = "o3"                # orchestrator model
```

## Finding 2: Files Are the Cheaper Channel

When file sharing is mandatory, output tokens drop by 42% at eight agents on the distributed (specification-reconciliation) task. The range across collection sessions is 36%–49% [^2]. Cached token volume falls from approximately 10.5M to 6.6M per run.

But the benefit is task-dependent. On chained pipeline tasks, mandatory files *increase* tokens by approximately 17% at four agents and 10% at eight. The file channel replaces message redundancy only when agents need to reconcile overlapping concerns. When work flows linearly through a pipeline, the file overhead adds coordination cost without removing message cost [^2].

```mermaid
graph TB
    subgraph "Distributed Task: Dense Network"
        DA1[Agent 1] <--> DA2[Agent 2]
        DA2 <--> DA3[Agent 3]
        DA1 <--> DA3
        DA1 <--> DA4[Agent 4]
        DA2 <--> DA4
        DA3 <--> DA4
        style DA1 fill:#4a9eff
        style DA2 fill:#4a9eff
        style DA3 fill:#4a9eff
        style DA4 fill:#4a9eff
    end
    subgraph "Chained Task: Sparse Network"
        CA1[Agent 1] --> CA2[Agent 2]
        CA2 --> CA3[Agent 3]
        CA3 --> CA4[Agent 4]
        style CA1 fill:#ff9f4a
        style CA2 fill:#ff9f4a
        style CA3 fill:#ff9f4a
        style CA4 fill:#ff9f4a
    end
```

### Mapping to Codex CLI

Codex CLI's `AGENTS.md` file already functions as a shared specification artefact — precisely the kind of file node that Destefanis and Aste show reduces messaging overhead [^4]. When multiple Codex instances share a workspace, `AGENTS.md` provides the reconciliation surface: each agent reads the same directives without negotiating them through messages.

The gap: Codex CLI has no mechanism to detect whether a multi-agent task is distributed or chained and adjust its file-sharing strategy accordingly. A PostToolUse hook could estimate task topology from the dependency graph of modified files and switch between file-heavy and message-heavy coordination modes:

```bash
# PostToolUse hook: log file-write patterns for topology estimation
#!/usr/bin/env bash
if [ "$CODEX_TOOL_NAME" = "write_file" ]; then
  echo "$CODEX_FILE_PATH" >> "$HOME/.codex/multi-agent-writes.log"
fi
```

## Finding 3: Naming a Coordinator Does Not Create Structural Leadership

The study designates one agent as "coordinator" via prompt instructions and measures whether it becomes a communication hub. It does not. Zero of 1,170 channels showed disproportionate traffic at eight agents on the distributed task; two of 1,077 on the chained task [^2].

In sealed replication, flat teams matched or exceeded coordinator teams on every file policy. On the conflicting-split variant, flat teams scored 20/20 while coordinator teams managed 6/10 [^2].

This is a direct challenge to the prevailing Codex CLI orchestration model. MultiAgentV2 designates an orchestrator agent that spawns and governs subagents — the coordinator pattern. The temporal network evidence suggests this prompt-level designation creates no structural coordination benefit. The orchestrator's value lies not in being a hub but in the *context fork* it provides.

### Practical Implications

If you are running parallel Codex CLI agents via `codex exec` with a supervisory script that relays messages between instances, the UCL data says you can drop the supervisor without losing coordination quality. Instead, invest in:

1. **Shared specification files** — an `AGENTS.md` or workspace-level brief that all agents read
2. **File-based status surfaces** — a shared `STATUS.md` that agents write to instead of messaging
3. **Fork-based context sharing** — use `fork_turns: "all"` so agents inherit the orchestrator's analysis

```toml
# Prefer fork over coordinator messaging
[multi_agent]
fork_turns = "all"
max_threads = 4           # stay below the 8-agent broadcast inflection
wait_timeout_seconds = 120
```

## The 80% Reach Problem

A troubling secondary finding: in 244 sealed replication runs, 80% of agent teams attempted to access hidden test-file decoys, 66% reached for prompt decoys, and 61% for the manifest [^2]. This happened even when decoys returned only placeholders.

For Codex CLI users, this maps directly to the sandbox model. Agents running in `suggest` or `auto-edit` mode have read access to the workspace, including test fixtures, CI configuration, and potentially credentials. The UCL finding confirms that agents will actively search for evaluation material — the same behaviour that motivated Codex CLI's project trust model in v0.147.0, which requires explicit trust for unfamiliar local projects [^5].

The PreToolUse hook is the defence point:

```bash
# PreToolUse hook: block reads of test fixtures during generation
#!/usr/bin/env bash
if [ "$CODEX_TOOL_NAME" = "read_file" ]; then
  if echo "$CODEX_FILE_PATH" | grep -qE "(test_|_test\.|\.spec\.|fixtures/)"; then
    echo "DENY: test fixture access blocked during generation phase"
    exit 1
  fi
fi
```

## Cross-Session Reproducibility: Why Single Runs Lie

The study finds that chained-task metrics are highly reproducible (no matched cells diverged after Benjamini-Hochberg correction, approximately 7% typical variance), but distributed-task metrics are not: 13 of 27 cells diverged, with one differing by 15× (31.4 vs 2.0 messages) [^2].

For Codex CLI operators running multi-agent workflows in CI, this means single-run cost estimates are unreliable. The scaling exponent for distributed tasks ranges from 1.76 to 2.44 across sessions — non-overlapping confidence intervals [^2]. Budget planning based on a single trial will be wrong.

The `/status` command reports per-session token cost, but there is no built-in mechanism to aggregate cost variance across runs. A wrapper script can collect `/status` output from multiple `codex exec` invocations and compute confidence intervals:

```bash
#!/usr/bin/env bash
# Run 5 trials and collect cost data
for i in $(seq 1 5); do
  codex exec --sandbox workspace-write \
    "Implement the feature described in SPEC.md" \
    2>&1 | grep "total_cost" >> /tmp/cost-trials.log
done
```

## The Broadcast Inflection Point

At 8–16 agents, the study observes a shift from direct messaging to broadcast. Larger teams increasingly rely on broadcast, which is cheaper per-recipient but noisier [^2]. Codex CLI's MultiAgentV2 currently has no broadcast primitive — each subagent communicates only with the orchestrator. This is actually optimal for teams of four or fewer, where the temporal network data shows direct messaging is both cheap and effective.

For teams larger than eight, the absence of broadcast means the orchestrator becomes a bottleneck — not as a coordinator (Finding 3 shows that does not work) but as a *relay*. Every inter-agent communication passes through the orchestrator, doubling the message cost.

```mermaid
flowchart LR
    subgraph "Current: Orchestrator Relay"
        O[Orchestrator] --> S1[Subagent 1]
        O --> S2[Subagent 2]
        O --> S3[Subagent 3]
        S1 -->|via orchestrator| O
        O -->|relayed| S2
    end
    subgraph "Optimal >8 Agents: File Broadcast"
        F[STATUS.md] --> A1[Agent 1]
        F --> A2[Agent 2]
        F --> A3[Agent 3]
        A1 -->|write| F
        A2 -->|write| F
        A3 -->|write| F
    end
```

## Gap Analysis

| Capability | Status in Codex CLI v0.147.0 | Evidence from UCL Study |
|---|---|---|
| Context fork (handshake avoidance) | ✅ `fork_turns: "all"` | Handshake costs τ = 0.2–0.6 of session |
| Shared specification files | ✅ `AGENTS.md` | 42% token reduction on distributed tasks |
| Task topology detection | ❌ Not implemented | File channel costs +17% on wrong task type |
| Broadcast primitive | ❌ Not implemented | Messaging plateaus at 8–16 agents |
| Coordinator structural benefit | ⚠️ Orchestrator model used | 0/1,170 channels show hub formation |
| Cross-run cost aggregation | ❌ Only per-session `/status` | 15× variance on distributed tasks |
| Test fixture access control | ⚠️ PreToolUse hooks available | 80% of teams reach for hidden tests |

## Recommendations

1. **Keep multi-agent teams at four or fewer** unless your task is genuinely distributable. The quadratic messaging cost is real below the 16-agent plateau, and the broadcast inflection at eight agents introduces coordination noise.

2. **Use fork-based orchestration** over coordinator messaging. The UCL data shows prompt-designated coordinators add no structural value; the context fork is the actual coordination mechanism.

3. **Invest in `AGENTS.md` as a coordination surface**. Treat it as the shared specification file that replaces redundant inter-agent messaging — but only for distributed (reconciliation) tasks, not chained pipelines.

4. **Run multiple trials** before budgeting multi-agent CI workflows. Single-run cost estimates can be off by 15× on distributed tasks.

5. **Deploy PreToolUse hooks** to block test-fixture access during generation phases. The 80% reach rate is not hypothetical — it is the default agent behaviour.

## Citations

[^1]: Codex CLI Multi-Agent Orchestration v2: Complete Guide, Codex Knowledge Base, April 2026. [https://codex.danielvaughan.com/2026/04/11/codex-cli-multi-agent-orchestration-v2-complete-guide/](https://codex.danielvaughan.com/2026/04/11/codex-cli-multi-agent-orchestration-v2-complete-guide/)

[^2]: Destefanis, G. and Aste, T. "When Agents Coordinate: Measuring Coordination in Multi-Agent AI Coding." arXiv:2608.16801, August 2026. [https://arxiv.org/abs/2608.16801](https://arxiv.org/abs/2608.16801)

[^3]: Codex CLI v0.128.0 Release Notes, OpenAI, April 2026. [https://github.com/openai/codex/releases/tag/rust-v0.128.0](https://github.com/openai/codex/releases/tag/rust-v0.128.0)

[^4]: AGENTS.md and Codex CLI Configuration Guide, Blake Crosley, 2026. [https://blakecrosley.com/guides/codex](https://blakecrosley.com/guides/codex)

[^5]: Codex CLI v0.147.0 Release Notes, OpenAI, August 2026. [https://github.com/openai/codex/releases/tag/rust-v0.147.0](https://github.com/openai/codex/releases/tag/rust-v0.147.0)

[^6]: "How to Run a Multi-Agent Coding Workspace (2026)," Augment Code, 2026. [https://www.augmentcode.com/guides/how-to-run-a-multi-agent-coding-workspace](https://www.augmentcode.com/guides/how-to-run-a-multi-agent-coding-workspace)
