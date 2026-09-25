---
title: "Agentic Workspace Taxonomy: From Terminal Tabs to Purpose-Built Orchestration, Codex CLI"
parent: "Articles"
nav_order: 1164
date: 2026-09-22T08:00:00+00:00
last_modified_at: 2026-09-25T09:05:33+01:00
tags: ["codex-cli", "multi-agent", "orchestration", "workflow", "productivity", "tooling", "agentic", "context-management"]
---

# Agentic Workspace Taxonomy: From Terminal Tabs to Purpose-Built Orchestration, Codex CLI


---

When Steve Yegge asked six hundred engineers which tool they use to manage multiple coding agents, the answers sorted into five distinct categories.[^1] Terminal tabs. Terminal multiplexers. Shell wrappers around queue commands. IDE extensions. Standalone TUI orchestrators. The spread was not random — each category represents a coherent set of trade-offs, and each breaks down at a predictable point of scale.

This article maps those categories into a structured taxonomy. For each tier, it identifies what the approach optimises for, which Codex CLI features it depends on, and the specific failure mode that appears as agent count or task complexity grows. The goal is a decision framework: given your current workload and team size, which tier fits, and when do you know you have outgrown it?

## Why the Taxonomy Matters

The absence of consensus in Yegge's thread is not a sign that the problem is unsolved. It is a sign that different practitioners are at different points in their scaling curve. The developer managing two or three agents intermittently has different constraints from the team running eight continuous overnight agents against a shared repository. A taxonomy makes that curve legible — and makes it easier to anticipate the next ceiling before hitting it.[^1]

The source material for this analysis draws on the context engineering and working-memory research that has emerged alongside multi-agent Codex CLI adoption.[^2][^3] The recurring finding is that workspace failures are almost never compute failures. They are coherence failures: the human loses track of which agent decided what, which tasks are blocking which, and which context is stale. The taxonomy tiers differ primarily in how much of that coherence they externalise into tooling.

---

## Tier 0: Raw Terminal Tabs

**Optimises for:** Zero setup time. Immediate start.

**Required Codex CLI features:** None beyond the default interactive TUI.

The raw terminal tab approach — one terminal window or tab per agent — is where most developers begin. It has genuine advantages at small scale: no configuration overhead, no abstraction layer between you and the agent's raw output, and complete flexibility over what runs in each pane. For a single developer running one or two exploratory agents on isolated tasks, it is often sufficient.

The failure mode is cognitive, not technical. Each open session is a context reconstruction problem: to assess an agent's status, you must switch to its terminal, read recent output, and rebuild your understanding of where it is and what it decided. With two sessions, this takes seconds. With five, it takes minutes. With ten, the reconstruction cost exceeds the parallelism benefit. Asay's InfoWorld report quotes engineers describing this ceiling as "managing the agents became the job."[^1]

There is no Codex CLI configuration that makes raw terminal tabs scale. The ceiling is architectural, not configurable.

---

## Tier 1: Terminal Multiplexer Layers (tmux / Zellij)

**Optimises for:** Visibility into multiple concurrent sessions without context reconstruction.

**Required Codex CLI features:** `tui.auto_recap`, named `--session` handles (v0.153.0+).

Terminal multiplexers organise the raw-tab chaos by placing sessions into panes within a persistent layout. A developer running six agents can structure their workspace as a 2×3 grid with each pane showing a live agent. The multiplexer layer itself — tmux, Zellij, or Wezterm — handles session persistence, so closing and reopening the terminal does not lose agent state.

The Codex CLI features that make this tier viable are `tui.auto_recap` and named session handles. Auto-recap generates a structured summary every N turns, so that glancing at a pane tells you the agent's current status without reading scroll-back. Named sessions allow you to disconnect from a pane, do other work, and reconnect to the correct session by name.

```toml
# config.toml
[tui]
auto_recap = true
auto_recap_interval = 10
```

Jacob Voytko's implementation — described in Yegge's thread as a shell layer over tmux — represents the upper end of what multiplexer layering can accomplish.[^1] He added automatic pane labelling with task tags and a status-bar script that polled `codex queue list` every thirty seconds to inject a compact queue summary.

The failure mode for this tier is the absence of a unified decision surface. Each pane is an independent view. If agent A reaches a decision point whose outcome affects agent B's current work, the multiplexer layer gives you no mechanism to surface that relationship. You notice it only when you happen to look at both panes, which — at six or more concurrent sessions — is an unreliable habit. The coherence debt model from Mohammadi et al. applies directly here: the agent may write an edit without the developer having registered that a blocking decision in another pane already resolved the question differently.[^3]

Multiplexer layers work reliably for three to six agents on independent tasks. They degrade when tasks have cross-dependencies and when session count exceeds comfortable visual span (typically six panes on a single monitor).

---

## Tier 2: Queue Wrappers

**Optimises for:** Structured task dispatch, status aggregation, and tag-based filtering.

**Required Codex CLI features:** `codex queue`, `codex queue list --tag`, `--format json` output (v0.153.0+).

Queue wrappers are shell scripts and functions that compose around `codex queue` to add filtering, sorting, and display capabilities that the native CLI does not provide. They transform the queue from a task-dispatch command into a lightweight task-management surface.

The decision-impact wrapper described in the companion article[^4] — `cq-impact`, which sorts queue output by `[IMPACT:HIGH]` preamble and surfaces blocked tasks first — is one pattern. Tag-based sprint views are another:

```bash
# In .bashrc / .zshrc
cq-sprint() {
  local tag="${1:-current-sprint}"
  codex queue list --tag "$tag" --format json \
    | jq -r '.tasks[] | "\(.status)\t\(.impact // "NORMAL")\t\(.id)\t\(.description[:55])"' \
    | sort -k2,2r -k1,1 \
    | column -t
}

# Usage
cq-sprint sprint-48-perf
```

Queue wrappers can also be composed with `watch` or a short polling loop to produce a persistent status display in a dedicated tmux pane — effectively a live dashboard that does not require switching context to check.

This tier requires disciplined task description hygiene: impact annotations, dependency preambles, and tag assignment must be written at dispatch time. If you forget to tag a task or omit its dependency metadata, the wrapper cannot surface it correctly. The quality of the output is bounded by the quality of the input.

The failure mode is that queue wrappers remain a pull interface. You query the queue; the queue does not push decisions to you. When an agent reaches a decision point that blocks downstream work, your wrapper tells you about it only on the next poll cycle. For teams running overnight agents or batch workloads spanning multiple hours, a polling interval of thirty seconds produces acceptable latency. For interactive development sessions where agents surface decisions every few minutes, the delay is perceptible and occasionally costly.

Queue wrappers scale to eight to twelve concurrent tasks before the wrapper's display becomes harder to parse than a well-organised tmux layout. At that point, the information density exceeds what a terminal column layout handles gracefully.

---

## Tier 3: IDE Extensions

**Optimises for:** Embedding agent status into an existing development surface without requiring a separate terminal workspace.

**Required Codex CLI features:** `codex queue list` JSON output, session reconnect (v0.153.0+), Guardian history.

IDE extensions — sidebar panels in VS Code or Cursor that display queue status, session summaries, and decision notifications — represent the first tier that integrates agent management into the developer's primary working context rather than alongside it. The appeal is attention economy: instead of switching to a terminal to poll queue status, the status is ambient in the sidebar.

Several extensions in the post-Yegge ecosystem follow this pattern. They call `codex queue list` on a timer, parse the JSON output, and render status badges alongside task descriptions. More sophisticated implementations surface Guardian approval requests as notification toasts, so the developer sees a pending destructive-action approval without leaving the editor.

The Codex CLI features required at this tier are stable JSON output formats and Guardian history persistence. Without stable JSON schemas, an IDE extension becomes a brittle integration that breaks on minor CLI version bumps. Guardian history (PRs #41879 and #42065 in v0.153.0) is what makes the extension's approval notifications meaningful — without it, the agent's approval request has no memory of the previous session's decision context.[^5]

The failure mode is the surface mismatch: IDE extensions are optimised for the single-developer workflow where agent output is ancillary to code editing. Teams running agents autonomously — without a developer actively editing in parallel — find that the IDE extension model forces them to keep an editor open as a passive monitoring terminal. At that point, a standalone TUI is a more appropriate fit.

IDE extensions also struggle with shared-team contexts. They are scoped to a single developer's session and do not expose a shared view of what other developers' agents are doing.

---

## Tier 4: Standalone TUI Orchestrators (Herdr, Conductor)

**Optimises for:** Unified multi-agent visibility, dependency-aware task scheduling, and team-shared state.

**Required Codex CLI features:** JSON output, `--session` handles, `AGENTS.md` dependency annotations, MCP server connections.

Standalone TUI orchestrators — Herdr, Conductor, and the category of tools they represent — are purpose-built for the agentic workspace problem. They are not layered on top of an existing terminal workflow; they replace the terminal as the primary human-agent interface.

The defining characteristic of this tier is push-based notification with dependency context. When an agent reaches a decision point, the orchestrator surfaces it not as a passive status change but as an active interrupt: here is the decision, here are the tasks that depend on its outcome, here is the context the agent has accumulated. The human makes the decision inside the orchestrator's interface, and the result is propagated to all dependent tasks.

Herdr, the most widely cited tool in Yegge's thread, implements dependency tracking by parsing AGENTS.md dependency annotation blocks — the same format described in the AGENTS.md orchestration manifest article[^4] — and constructing a live task graph. When task A completes, Herdr automatically unblocks tasks that listed A in their `depends_on` field. When a decision in task A contradicts a constraint in task B's annotation, Herdr flags the conflict before the dependent agent acts on it.

This tier requires the most Codex CLI infrastructure: stable session handles, structured AGENTS.md dependency blocks, and either a direct MCP connection or a polling adapter against `codex queue list`. It also requires upfront discipline — the task graph must be specified before the batch starts, not reconstructed after the fact.

The failure mode for standalone orchestrators is setup cost relative to task size. For a developer running five ad-hoc tasks in an afternoon, configuring Herdr's project file and populating dependency annotations adds thirty minutes of overhead for ten minutes of benefit. The orchestrator tier pays off on workloads that are repeated, structured, and run at a scale where human monitoring is the genuine bottleneck — typically four or more concurrent agents with cross-dependencies, run on a recurring schedule.

---

## Choosing a Tier

The following decision criteria cover the majority of common workloads:

| Agent count | Task structure | Recommended tier |
|---|---|---|
| 1–2 | Independent, ad hoc | Tier 0 (raw tabs) |
| 3–6 | Independent, parallel | Tier 1 (multiplexer) |
| 4–10 | Independent, batch | Tier 2 (queue wrappers) |
| 3–8 | Active editing, single developer | Tier 3 (IDE extension) |
| 4+ | Cross-dependent, recurring | Tier 4 (standalone TUI) |

The signal that you have outgrown a tier is not task failure — it is reconstruction cost. When you find yourself spending more than a few minutes per decision point rebuilding context about what the agent decided and why, the current tier is no longer handling coherence. Moving up one tier is the correct response; moving to a higher-bandwidth model is not.[^3]

A final note on Codex CLI configuration that applies at every tier: `tui.auto_recap`, named sessions, and Guardian history are the three settings most likely to reduce reconstruction cost at any scale. If you have not configured them, do that first. Many of the problems that appear to require a higher-tier tool disappear once the agent's own context summaries are reliable.

---

## Footnotes

[^1]: Asay, M. (2026, September 15). *The best IDE for agentic AI may not be an IDE at all*. InfoWorld. https://www.infoworld.com/article/4221481/the-best-ide-for-agentic-ai-may-not-be-an-ide-at-all.html
[^2]: Böckeler, B. (2026). *Harness engineering*. martinfowler.com. https://martinfowler.com/articles/harness-engineering.html
[^3]: Mohammadi, B., Klein, L., Chadha, A., Arora, A., & Bindschaedler, L. (2026). *The working set of a coding agent: Coherence debt in repository-scale tasks*. arXiv:2608.16630. https://arxiv.org/abs/2608.16630
[^4]: Vaughan, D. (2026, September 21). *Decision-impact scoring for Codex queue: Task interdependency metadata, Codex CLI*. codex-resources. https://danielvaughan.github.io/codex-resources/articles/2026-09-21-decision-impact-scoring-codex-queue-task-interdependency-metadata-codex-cli
[^5]: OpenAI. (2026). *Codex CLI v0.153.0 release notes — Guardian history persistence (PRs #41879, #42065)*. https://github.com/openai/codex/releases/tag/rust-v0.153.0. Retrieved 2026-09-22.
