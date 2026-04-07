---
title: "Codex CLI Subagent Gotchas: Known Issues and Workarounds (March 2026)"
parent: "Articles"
nav_order: 88
---

# Codex CLI Subagent Gotchas: Known Issues and Workarounds (March 2026)


If you're building agentic pod workflows with Codex CLI subagents, here are the practical bugs and undocumented behaviours you'll hit — and how to work around them. Sourced from open GitHub issues as of March 2026.

---

## 1. Custom TOML Agents Aren't Invokable by Name in Tool-Backed Sessions

**Issue:** [#15250](https://github.com/openai/codex/issues/15250) *(~1 week old as of 2026-03-29)*

The docs imply you can store custom agents as `.codex/agents/*.toml` files and have them invoked by name during a session. In practice, the `spawn_agent` tool call only accepts `agent_type` plus explicit `prompt`/`model` overrides — there's no parameter to say `"spawn agent defined in .codex/agents/security-reviewer.toml"`.

**Workaround:**
```markdown
Read the TOML file first, then inject its `developer_instructions` as prompt overrides when spawning a generic worker:

"Spawn a worker subagent with the following instructions: [paste contents of security-reviewer.toml's developer_instructions field]"
```

This is verbose but functional. Keep TOML `developer_instructions` short and precise for this reason.

**Impact:** High — anyone building reusable named subagent personas will hit this.

---

## 2. Subagents Getting Stuck as "Awaiting Instruction"

**Issue:** [#14866](https://github.com/openai/codex/issues/14866) *(~2 weeks old)*

Symptoms: Spawned subagents enter an "awaiting instruction" state and don't proceed. Looking at session JSONL logs reveals 62 `spawn_agent` entries but only 15 `close_agent` entries — a strong signal of a resource leak.

**Root causes identified:**
- Subagent threads exceeding the max 6 concurrent thread limit (Issue #11999)
- `inotify` watch limit exhausted on Linux — Codex silently reports "not found" instead of forwarding the real error
- Non-deterministic model selection when `model` is omitted from agent TOML

**Workarounds:**
```toml
# config.toml — set explicit limits
[agents]
max_threads = 4      # stay below the 6-thread wall with margin
max_depth = 1        # prevent recursive fan-out

# Linux: increase inotify limit if running many parallel sessions
# echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf
```

Always explicitly close subagent threads in your prompts: *"After each subagent returns, close it before spawning the next batch."*

---

## 3. `spawn_agent(model="gpt-5.4-mini")` Shows Wrong Model in Child Thread

**Issue:** [#15177](https://github.com/openai/codex/issues/15177) *(March 19, 2026)*

When you explicitly spawn a subagent with `model: "gpt-5.4-mini"`, the child thread's UI and metadata record `gpt-5.4` instead. The agent **does** run on mini (cost is correct), but the display lies.

**Impact:** Medium — misleading if you're auditing which model handled which task. Not a functional bug.

**Workaround:** Log model usage via hooks rather than relying on thread metadata for cost attribution:

```bash
# ~/.codex/hooks/sessionstop.sh — log session cost from JSONL
jq '.model, .usage' ~/.codex/sessions/*.jsonl | tail -20
```

---

## 4. Approval Surfacing From Inactive Subagent Threads

When many agents run in parallel, approval requests can surface from inactive threads while you're viewing the main thread. The overlay shows the source thread label — press `o` to open that thread and review context before approving, rejecting, or answering.

In non-interactive mode (`codex exec`), any action requiring new approval causes an error back to the parent. **Design non-interactive subagent workflows so workers operate within their inherited sandbox with no fresh approvals needed.** Use `--sandbox workspace-write` and constrain work scope in the prompt.

---

## 5. Guardian Subagent for Smart Approvals

Instead of blanket `--yolo` approval in multi-agent workflows, use the `guardian_subagent` role for risk-based routing:

```toml
# .codex/agents/guardian.toml
name = "guardian"
description = "Reviews pending tool approvals with context before deciding"
developer_instructions = """
You are a security-conscious approval guardian. For each pending tool call:
1. Read the full context of why this action is being requested
2. Apply a risk score: LOW (auto-approve), MEDIUM (approve with note), HIGH (escalate to user)
3. Block actions that modify files outside the project directory, make network requests to unknown hosts, or delete data

Err on the side of escalation for ambiguous cases.
"""
model = "gpt-5.4-mini"   # Fast enough for approval decisions; saves quota
```

Configure in `config.toml`:
```toml
[agents]
approval_guardian = "guardian"
```

This gives you the safety of manual approval with the speed of automation for low-risk actions.

---

## 6. Model Selection Strategy for Subagents (GPT-5.4 Mini)

As of March 2026, `gpt-5.4-mini` is purpose-built for subagent roles:
- **Quota:** 30% of a full GPT-5.4 turn — you get ~3x throughput
- **Benchmark:** 54.4% SWE-Bench Pro (vs 57% for GPT-5.4 — only 3 points behind)
- **Best for:** Exploration, large-file review, parallel read tasks, approval routing

Recommended routing for agentic pod:

| Task type | Model |
|-----------|-------|
| Architecture planning, complex reasoning | `gpt-5.4` or `gpt-5-codex` |
| Parallel file analysis, codebase scans | `gpt-5.4-mini` |
| Approval routing (guardian) | `gpt-5.4-mini` |
| Real-time interactive coding | `gpt-5.3-codex-spark` (Pro only) |
| Classification, filtering | `gpt-5.4-nano` (API only) |

**Note:** Until Issue #15177 is fixed, the thread UI will show `gpt-5.4` even when mini was requested. Verify actual usage via session JSONL.

---

## 7. Sub-Agent Completes But Returns Handoff Failure Instead of Payload

**Issue:** [#16051](https://github.com/openai/codex/issues/16051) *(opened March 27, 2026 — open)*

Symptoms: A sub-agent spawned with `spawn_agent` or via `multi_agent_v2` `assign_task` reaches completion status, but instead of receiving the expected result payload the parent agent receives a meta-level failure message:

> *"The sub-agent handoff did not complete normally in this run. It accepted the task envelope, but it never returned the requested audit payload back into the session."*

The sub-agent **did** execute — the task was completed — but the result-delivery plumbing between the child and parent threads is broken in this code path. Likely a regression introduced in v0.118.0-alpha.2.

**Workaround:**
None confirmed. The GitHub bot flagged potential duplicates at issues #14318, #14667, #14866, suggesting result delivery has had intermittent failures across several releases.

Practical mitigation while open:
- Have the sub-agent write its output to a file (`/tmp/subagent-output-{name}.json`) and have the parent read the file rather than waiting for the result in the return payload.
- This file-based rendezvous is more robust than relying on the handoff channel anyway — it leaves an artefact even if the session crashes.

**Impact:** High for v0.118.0-alpha users building multi-agent pipelines. Not present in stable v0.117.0.

---

## Checklist: Before Running Multi-Agent Workflows

- [ ] Set `max_threads = 4` and `max_depth = 1` in config.toml
- [ ] Confirm workers have `--sandbox workspace-write` scope (not `--yolo`)
- [ ] Pre-define any custom agent personas as explicit prompt blocks, not TOML file references
- [ ] Explicitly tell the parent to close subagents after receiving results
- [ ] On Linux: verify inotify limits (`cat /proc/sys/fs/inotify/max_user_watches`)
- [ ] Use `gpt-5.4-mini` for parallel scan/review workers to preserve quota
- [ ] **Stick to stable v0.117.0** for production agentic workflows — v0.118.0 alphas have an open handoff failure bug (#16051)

---

*Source: GitHub issues openai/codex #15250, #14866, #15177, #11999, #16051 — researched 2026-03-29, updated 2026-03-31*
