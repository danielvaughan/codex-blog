---
title: "Decision-Impact Scoring for Codex Queue: Encoding Task Interdependency Before You Launch the Batch"
parent: "Articles"
nav_order: 1163
date: 2026-09-21T15:00:00+00:00
last_modified_at: 2026-09-22T11:39:29+01:00
tags: ["codex-cli", "multi-agent", "codex-queue", "task-management", "agents-md", "decision-impact", "orchestration", "workflow"]
---

# Decision-Impact Scoring for Codex Queue: Encoding Task Interdependency Before You Launch the Batch


---

The missing primitive in `codex queue` is not a smarter scheduler. It is a way to express, before the batch starts, which tasks produce decisions whose consequences cascade. The September 2026 survey of six hundred engineers described a common failure pattern: agents completing work that requires a human choice, that choice sitting unattended while downstream agents continue in a direction that will need reversing, and the developer notified too late to prevent the rework.[^1] The failure is not that agents move too fast. It is that the queue has no way to know which completed task needs attention first.

Decision-impact scoring is the discipline of making that information explicit before the batch runs, embedding it in the task description and in AGENTS.md in a form that both the developer and the running agents can read. It requires no new tooling. It requires a different habit at task-creation time.

---

## The Gap That Custom Dashboards Are Filling

When Yegge's thread asked which workspace tools engineers had built or adopted for managing parallel agents, the answers converged on one underlying need: filtered notification.[^1] Not all completions. Not all questions. The specific completion that unlocks or blocks something else.

Herdr, Conductor, and the tmux-based shell wrappers described in the thread all attempt to provide this. They instrument the agent outputs, tag completions by type, and surface the ones that have downstream dependencies. They are solving, in bespoke tooling, a problem that should be expressible in the task description itself.

`codex queue` stores the task description with the result. If that description contains structured dependency metadata, any tool — including a ten-line shell function — can parse it and produce the prioritised view that developers are currently building custom dashboards to see.

---

## A Structured Task Description Format

The task description passed to `codex queue` is free text, but agents read it in full. A structured preamble that encodes impact metadata is readable by both humans and agents, and it costs one minute to write per task.

```bash
codex queue "[IMPACT:HIGH][BLOCKS:api-contract,integration-tests] \
  Investigate and resolve the authentication token expiry bug in payments-service. \
  Decision required: confirm whether to extend token TTL or introduce refresh token rotation. \
  This choice determines the API contract shape for downstream tasks."

codex queue "[IMPACT:MEDIUM][DEPENDS-ON:auth-bug-fix] \
  Draft the updated OpenAPI spec for the payments endpoint once auth decision is confirmed. \
  No human decision required — proceed to completion."

codex queue "[IMPACT:LOW][DEPENDS-ON:openapi-spec] \
  Generate client SDK stubs from the finalised OpenAPI spec. \
  No human decision required — proceed to completion."
```

The three fields that matter are `IMPACT` (HIGH / MEDIUM / LOW), `BLOCKS` (a comma-separated list of task slugs this completion must precede), and `DEPENDS-ON` (a list of task slugs that must complete before this task can proceed). `IMPACT:HIGH` indicates that a decision made by this agent will be read by other agents or will shape the human's next action in a way that compounds if delayed.

Agents reading a `DEPENDS-ON` task check the queue status of its dependencies before beginning substantive work. An agent that encounters unresolved `DEPENDS-ON` entries writes a brief note to `tmp/agent-log.md` and exits cleanly rather than proceeding on stale assumptions.[^2]

---

## AGENTS.md as the Dependency Register

The task description carries per-task metadata. AGENTS.md carries the batch-level dependency register — the document that gives any agent in the session a full view of the task graph before it starts work.

```markdown
## Batch: Payments Auth Refactor — 2026-09-21

### Impact Register

| Task Slug         | Impact | Status      | Blocks                          | Depends On        |
|-------------------|--------|-------------|----------------------------------|-------------------|
| auth-bug-fix      | HIGH   | IN_PROGRESS | api-contract, integration-tests | —                 |
| openapi-spec      | MEDIUM | BLOCKED     | sdk-stubs, e2e-tests            | auth-bug-fix      |
| sdk-stubs         | LOW    | BLOCKED     | e2e-tests                       | openapi-spec      |
| integration-tests | MEDIUM | BLOCKED     | —                               | auth-bug-fix      |
| e2e-tests         | LOW    | BLOCKED     | —                               | sdk-stubs         |

### Decision Points

- **auth-bug-fix**: Agent will surface TTL vs refresh-token choice. Human must confirm before `openapi-spec` begins.
- **openapi-spec**: Agent may proceed to completion once `auth-bug-fix` status is COMPLETE.
- All other tasks: No human decision required.
```

The impact register is the pre-flight document you complete before running `codex queue`. Writing it forces the decision-point analysis that developers otherwise defer until agents are mid-flight and the cost of a wrong turn is already accumulating. The Overnight Agent research found that the most consequential gaps between harness specification and production behaviour were ones that could have been identified at harness-design time — the three-queue-versus-eighteen-queue discrepancy that produced incorrect retry logic, for example, was an assumption that a dependency register would have surfaced as a question before any agent ran.[^3]

---

## A Shell Wrapper for Prioritised Queue Status

With structured task descriptions, a short shell function can parse the queue output and display tasks in impact order rather than recency order:

```bash
# In your .bashrc or .zshrc
cq-impact() {
  local tag="${1:-}"
  local list_cmd="codex queue list --format json"
  [[ -n "$tag" ]] && list_cmd="$list_cmd --tag $tag"

  eval "$list_cmd" | jq -r '
    .tasks[]
    | {
        id: .id,
        status: .status,
        impact: (
          if (.description | test("\\[IMPACT:HIGH\\]")) then "HIGH"
          elif (.description | test("\\[IMPACT:MEDIUM\\]")) then "MEDIUM"
          else "LOW"
          end
        ),
        blocks: (
          .description | capture("\\[BLOCKS:(?<b>[^\\]]+)\\]") | .b // "—"
        ),
        desc: (.description | ltrimstr("[IMPACT:HIGH]") | ltrimstr("[IMPACT:MEDIUM]") | ltrimstr("[IMPACT:LOW]") | .[0:55])
      }
    | [.impact, .status, .id, .blocks, .desc]
    | @tsv
  ' | sort -k1,1 -k2,2 | column -t -s $'\t'
}
```

Running `cq-impact` produces a view sorted by impact level, then by status. HIGH-impact tasks that have reached `AWAITING_INPUT` appear at the top. LOW-impact tasks that have completed without requiring a decision appear at the bottom. The developer sees immediately which completion requires action and what it blocks.

For teams already using `codex queue --tag` to group tasks by sprint milestone, adding impact metadata to the description is a lightweight extension of an existing habit rather than a new workflow layer.[^1]

---

## The Token Budget Interaction

Decision-impact scoring has a second function beyond notification prioritisation: it shapes token budget allocation. The rollout token budget feature distributes a shared ledger across all sub-agent threads in a session.[^4] Without impact metadata, the budget is consumed on a first-come basis — the LOW-impact task that started slightly earlier will draw down the same ledger as the HIGH-impact task that just surfaced a blocking decision.

With impact metadata in AGENTS.md, a session profile can assign `output_token_limit` differentially:

```toml
# config.toml
[session_profiles.high-impact]
model = "codex-1"
output_token_limit = 32000
approval_policy = "unless-allow-listed"

[session_profiles.low-impact]
model = "codex-mini-latest"
output_token_limit = 8000
approval_policy = "auto-edit"
```

```bash
# Launch with impact-matched profiles
codex queue --profile high-impact "[IMPACT:HIGH]..."
codex queue --profile low-impact "[IMPACT:LOW]..."
```

The HIGH-impact task gets the model and token budget appropriate to a decision with cascading consequences. The LOW-impact task gets a cheaper, faster model whose errors are recoverable. Token budget governance and decision-impact governance solve the same underlying problem from different directions: preventing undifferentiated concurrency from treating all tasks as equivalent when they are not.[^4]

---

## The Pre-Batch Discipline

The impact register and structured task descriptions are not features of `codex queue` — they are a pre-flight discipline. They require five to ten minutes of dependency analysis before launching a batch. For most multi-agent batches, this analysis reveals one or two decision points whose delay would have cost an hour of rework. That return justifies the time.

The habit to build is: before running `codex queue` for any batch of more than two tasks, write the impact register in AGENTS.md, tag each task description with `[IMPACT:]` and `[BLOCKS:]`, and identify explicitly which tasks require human decisions and which can proceed to completion. Agents that read AGENTS.md on startup have the full task graph available. The developer who returns to the queue twenty minutes later has a prioritised view of which completion needs attention first.

Decision-impact scoring is not a replacement for purpose-built orchestration tooling. It is the minimum viable encoding that makes the existing `codex queue` infrastructure significantly more useful without waiting for that tooling to mature.[^1]

---

[^1]: Yegge, S. (2026). "Which IDE do you use to manage multiple coding agents?" [Survey thread, 600+ responses]. Retrieved September 2026. Referenced in: "The Best IDE for Agentic AI Is Not an IDE" (codex-resources, September 16, 2026).

[^2]: Vaughan, D. (2026). "AGENTS.md as Multi-Agent Orchestration Manifest: Dependency Annotations, Exclusion Zones, and Rollback Conditions." codex-resources, September 21, 2026.

[^3]: Janusevicius, E. (2026). "We Let AI Agents Rewrite a 92M-Message-a-Day Service in Go. Zero Incidents." Checkly Engineering Blog, September 2026. Referenced in: "The Overnight Agent" (codex-resources, September 10, 2026).

[^4]: OpenAI. (2026). Codex CLI rollout token budget — pull requests #28746, #28494, #28707, #29423, #29324. GitHub, June 2026. Referenced in: "Rollout Token Budgets and Multi-Agent Delegation" (codex-resources, September 9, 2026).
