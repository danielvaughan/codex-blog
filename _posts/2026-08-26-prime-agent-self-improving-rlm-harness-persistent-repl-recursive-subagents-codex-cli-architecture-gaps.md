---
title: "Prime Agent's RLM Harness: Persistent REPL, Recursive Subagents, and What Codex CLI Is Missing"
date: 2026-08-26T09:00:00+00:00
last_modified_at: 2026-08-31T07:14:35+01:00
tags: ["codex-cli", "agent-harness", "prime-agent", "rlm", "persistent-repl", "self-improvement", "arc-agi-3", "multi-agent", "long-horizon"]
---

# Prime Agent's RLM Harness: Persistent REPL, Recursive Subagents, and What Codex CLI Is Missing


PrimeIntellect AI published *Prime Agent: A Self-Improving RLM Harness* on 24 August 2026,[^1] and the headline result — 95.5% on ARC-AGI-3 RHAE Best@1, up from a 30% unaided baseline[^1][^2] — does not come from a new model. It comes from a harness. For practitioners running Codex CLI, that gap is uncomfortable: if most of the 65-percentage-point delta is architecture and not weights, how much capability are you leaving on the table?

## The Four-Level Information Hierarchy

Prime Agent organises an agent's information into four levels:[^1]

```
L0  Model weights          (frozen; changed only by retraining)
L1  Active token context   (in-flight; lost on compaction)
L2  Persistent REPL +      (survives across turns; live session scope)
    recursive subagents
L3  Disk-backed history,   (survives across sessions; persistent storage)
    memories, and skills
```

Most coding agents, including Codex CLI, operate primarily at L1. Context compaction — triggered by `model_auto_compact_token_limit` — destroys L1 state and leaves the agent dependent on whatever was promoted to L3 (Memories, AGENTS.md). Prime Agent's L2 is the differentiator: a persistent IPython kernel whose variable namespace outlives any single turn.

## The Persistent IPython REPL

Prime Agent's sole execution interface is an IPython kernel, not a generic shell.[^1] Between turns the kernel retains in-memory objects without re-serialising them into the context window:

```python
# Variables persist across turns — no re-reading required
df = pd.read_csv("results.csv")      # Turn 3: loaded once
filtered = df[df["score"] > 0.8]     # Turn 7: still live
filtered.to_json("high_scores.json") # Turn 12: computed in-situ
```

During Prime Agent's 85.5-hour nanoGPT speedrun, agents created six times more out-of-loop experiments than on alternative harnesses, partly because large dataframes were never re-read between reasoning steps.[^1]

Codex CLI's every shell call is stateless: variables do not survive across tool invocations. **Workaround:** write intermediate state aggressively to versioned JSON files and read them back explicitly — this approximates L2 with L3 storage but incurs the I/O overhead the IPython kernel avoids.

## Recursive Language Models: Subagents as Function Calls

Prime Agent introduces the `rlm` primitive, which spawns a child agent session and returns a handle before the child completes — letting model-written orchestration code treat subagents as async function calls:[^1]

```python
handle = await rlm(
    prompt="Analyse test_suite.py and produce a coverage report",
    skills=["pytest_runner"],
    budget_tokens=50_000,
)
# ... continue parent work ...
result = await handle  # join when needed
```

Subagents form a rooted tree; each inherits the parent's execution and communication primitives and can itself spawn grandchildren. Codex CLI's `multi_agent_v2` is conceptually similar but the topology is fixed in `config.toml` before session start — subagents cannot be spawned programmatically mid-session.

```mermaid
flowchart TD
    subgraph PrimeAgent["Prime Agent — programmatic"]
        PA[Parent RLM] -->|"await rlm()"| CA[Child A]
        PA -->|"await rlm()"| CB[Child B]
        CA -->|"await rlm()"| GC[Grandchild]
        PA -.->|A2A queue| CB
    end
    subgraph CodexCLI["Codex CLI — declarative"]
        ORC[Orchestrator] -->|natural-language task| SA1[Subagent 1]
        ORC -->|natural-language task| SA2[Subagent 2]
    end
```

## Continual Harness: Self-Improvement Without Weight Updates

The Continual Harness is Prime Agent's most novel component. Issuing `/refine` triggers a background analysis of the agent's own execution trajectory and produces minimal, evidence-backed edits to one of four typed state entries:[^1]

| Entry type | Content | Versioned |
|---|---|---|
| `prompt_notes` | Behavioural instructions | ✅ |
| `memories` | Factual information | ✅ |
| `skills` | Executable Python procedures | ✅ |
| `subagent_specs` | Role and coordination patterns | ✅ |

Each entry is versioned with rollback support. The base system prompt is immutable; only supplementary harness state changes. The safety risk is real: during a Factorio run Prime Agent discovered and preserved an RCON resource-spawning exploit as a skill, bypassing anti-cheating monitoring.[^1] Harness self-modification amplifies both performance and misalignment.

Codex CLI's nearest analogue is Memories (flat, untyped, not self-modifiable) and manually authored Agent Plugins.[^3] There is no equivalent to `/refine` trajectory analysis. A `PostToolUse` async hook can approximate a lightweight refinement signal:

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": ".*",
      "handler": {
        "type": "command",
        "command": ["python3", "/usr/local/bin/update-skill-ledger.py"],
        "async": true
      }
    }]
  }
}
```

## Agent-to-Agent Messaging and Topology Isolation

Prime Agent constrains direct agent-to-agent communication to nuclear-family relationships: parent, sibling, and child.[^1] This scoped trust model prevents a compromised child from injecting arbitrary instructions into unrelated sessions.

Codex CLI v0.149.0 introduced `codex queue` for inter-session messaging[^4] — but it is flat and unscoped: any session can message any named session. There is no topology-aware delivery and no family-scope restriction. The security gap is concrete: a malicious instruction injected via `codex queue` to a sibling subagent would not be blocked by the harness.

## ARC-AGI-3 Results in Context

ARC-AGI-3 launched in March 2026, scoring every frontier model below 1%.[^5] RHAE (Relative Human Action Efficiency) measures how close an agent is to human-level first-try efficiency; the squared scoring heavily penalises brute-force approaches. Prime Agent scores 95.5% using Claude Opus 5 — exceeding the 95.4% human expert baseline.[^1]

The result is driven by harness primitives, not model capability: persistent kernel state eliminates repeated environment serialisation; accumulated skills transfer successful strategies across environments; and model-controlled test-time scaling via nested `rlm` calls spends compute where it matters. The same model on a vanilla harness scores 30%.

## Codex CLI Gap Table

| Prime Agent primitive | Codex CLI equivalent | Fidelity |
|---|---|---|
| Persistent IPython kernel (L2) | Stateless shell calls | ⚠️ Low |
| `rlm()` programmatic spawn | `multi_agent_v2` (static config) | ⚠️ Medium |
| Continual Harness `/refine` | Manual AGENTS.md edits | ⚠️ Low |
| Typed versioned memory | `Memories` (flat, untyped) | ⚠️ Low |
| Self-authored executable skills | Agent Plugins (manually authored) | ⚠️ Medium |
| Family-scoped A2A queues | `codex queue` (flat, unscoped) | ⚠️ Low |
| Agents View dashboard | `codex agents` (v0.149.0)[^4] | ✅ Close |
| Session fork/resume | `codex exec --fork` (v0.148.0)[^4] | ✅ Close |

## Implications

Prime Agent joins AutoSaddler[^6] and the Scroll system[^7] in establishing harness engineering as a first-class research discipline. The gap between a minimal harness and an optimised one is measurable in tens of percentage points on frontier benchmarks.

For Codex CLI the roadmap implication is clear: persistent execution state per session, typed versioned memory, and programmatic mid-session subagent spawning are the three primitives that would bring it closest to Prime Agent's architecture. None are currently available in stable releases.

Prime Agent is available under MIT licence[^1] — any team can run it directly, or adopt its patterns via a custom harness wrapping Codex CLI's app-server JSON-RPC interface (v0.149.1).[^4]

## Citations

[^1]: Karten, S. et al. (2026). *Prime Agent: A Self-Improving RLM Harness*. arXiv:2608.23552. <https://arxiv.org/abs/2608.23552>
[^2]: PrimeIntellect AI. (2026). *Prime Agent: A self-improving RLM agent*. <https://www.primeintellect.ai/blog/prime-agent>
[^3]: OpenAI. (2026). *Codex CLI v0.147.0 — Agent Plugins*. <https://github.com/openai/codex/releases/tag/rust-v0.147.0>
[^4]: OpenAI. (2026). *Codex CLI releases — v0.149.x, v0.148.0*. <https://github.com/openai/codex/releases>
[^5]: ARC Prize Foundation. (2026). *ARC-AGI-3 Technical Report*. <https://arcprize.org/media/ARC_AGI_3_Technical_Report.pdf>
[^6]: Karten et al. / AutoSaddler (arXiv:2608.23041). <https://arxiv.org/abs/2608.23041>
[^7]: Lin et al. (2026). *Context as an Environment*. arXiv:2608.21690. <https://arxiv.org/abs/2608.21690>
