---
title: "AGENTS.md as Multi-Agent Orchestration Manifest: Dependency Annotations, Exclusion Zones, and Rollback Conditions"
parent: "Articles"
nav_order: 1162
date: 2026-09-21T13:00:00+00:00
last_modified_at: 2026-09-26T10:24:07+01:00
tags: ["agents-md", "multi-agent", "orchestration", "codex-cli", "coordination", "workflow", "parallel-agents"]
---

# AGENTS.md as Multi-Agent Orchestration Manifest: Dependency Annotations, Exclusion Zones, and Rollback Conditions


---

AGENTS.md was designed as a per-session instruction set: a file that tells a coding agent how to behave in a given repository. What it has become, in teams running four or more parallel agents, is something different — a coordination substrate whose absence produces the category of failures that no individual agent configuration can prevent.

The most recent signal of this shift came from Steve Yegge's question to six hundred engineers: which IDE do you use to manage multiple coding agents?[^1] The answers diverged completely, but the failure modes described converged. Agents stepping on each other's migrations. Background agents committing to shared branches mid-refactor. One agent's successful merge invalidating the assumptions of three agents that were mid-task. These are not model failures. They are coordination failures, and they happen at the layer between agents, not inside any individual agent's context.

AGENTS.md sits at exactly that layer. A team that has invested in its AGENTS.md Playbook — tool definitions, coding standards, hook configurations, model profiles — has already made the file a first-class engineering artefact.[^2] Extending it to carry coordination semantics requires four additions: dependency annotations, milestone tags, exclusion zones, and rollback conditions. None of these require new tooling. They are conventions that agents can read and respect today.

## Why Per-Session Instructions Are Not Enough

A single Codex CLI session reads AGENTS.md on startup and uses it throughout the session. In a single-agent workflow, this is sufficient. The agent knows what it should do, what it must not do, and how to handle ambiguous cases.

In a multi-agent workflow, the relevant questions are different. Not "what should this agent do?" but "what is this agent allowed to do right now, given what the other agents are currently doing?" Not "how should this agent handle an error?" but "when this agent encounters a precondition failure, what should the team-level state be before it tries again?"

Per-session instructions cannot answer these questions because they describe agent behaviour in isolation. Coordination semantics describe agent behaviour in relation to shared state. AGENTS.md is already the most natural place to express both.

The compound constraint research reinforces why this separation matters.[^3] When agents accumulate instructions that interact with each other, the performance impact is not linear — combinations degrade capability at rates that individual instructions do not predict. An AGENTS.md file that mixes per-session instructions with ad hoc coordination hints (comments like "don't touch migrations while the data-team agent is running") creates exactly this combinatorial hazard. Structured coordination sections, separated from instruction sections, make the boundary explicit and reduce instruction interference.

## Dependency Annotations

A dependency annotation records which tasks must complete before other tasks can begin. This is the equivalent of a Makefile dependency graph, expressed in natural language that both humans and agents can read.

```markdown
## Agent Task Dependencies

- [ ] task: schema-migration
  depends_on: []
  owned_by: agent/db-agent
  status: IN_PROGRESS

- [ ] task: api-endpoint-update
  depends_on: [schema-migration]
  owned_by: agent/api-agent
  status: BLOCKED

- [ ] task: integration-tests
  depends_on: [schema-migration, api-endpoint-update]
  owned_by: agent/test-agent
  status: BLOCKED

- [ ] task: documentation-update
  depends_on: [api-endpoint-update]
  owned_by: agent/docs-agent
  status: BLOCKED
```

Each agent session starts by reading this section and checking whether its assigned task's dependencies are satisfied. If not, it records a reason and exits cleanly rather than proceeding on stale assumptions. An agent that encounters `BLOCKED` on its task writes a log entry to a shared location — `tmp/agent-log.md` works well — and terminates. The operator or orchestrating agent reviews the log and relaunches when upstream tasks complete.

This pattern eliminates the category of failure where an agent performs correct work on an incorrect state. The schema migration that the API agent assumed was complete, but was not, is now a visible constraint rather than a silent precondition.

## Milestone Tags

Milestone tags create shared checkpoints that agents use to synchronise their understanding of progress without requiring human intervention at each step.

```markdown
## Milestones

- [x] MILESTONE:db-schema-stable — Schema v4 applied and verified. All agents may read from new columns. Do not write to deprecated columns.
- [ ] MILESTONE:api-contract-frozen — OpenAPI spec finalised. Do not modify endpoint signatures or response shapes after this milestone.
- [ ] MILESTONE:feature-flag-enabled — `FLAG_NEW_CHECKOUT` enabled in staging. Integration test agents should use staging environment from this point.
```

An agent that completes work required for a milestone updates the checkbox and writes a brief note. Downstream agents check for the milestone before beginning work that depends on it. The milestone tag is not an automated lock — it is a convention — but conventions respected by every agent in a session profile are operationally equivalent to automated locks for most coordination purposes.

The key property of a milestone tag is that it is binary and persistent. Unlike a status field that might cycle through IN_PROGRESS, FAILED, RETRYING, a milestone is set once and not unset. This makes it safe to read without acquiring any form of lock, which matters when agents are running in parallel and reading AGENTS.md concurrently from separate processes.

## Exclusion Zones

An exclusion zone declares which parts of the codebase, which operations, or which external resources must not be accessed by more than one agent at a time.

```markdown
## Exclusion Zones

The following resources are serialised — only one agent may hold them at a time.
An agent that needs to work in an exclusion zone must:
1. Check `tmp/locks/` for an existing lock file named after the zone.
2. If no lock file exists, create one with its session ID and task name before proceeding.
3. Delete the lock file when work is complete or on any exit (clean or error).

Zones:
- ZONE:database-migrations — Any file in `db/migrations/`. Lock name: `db-migrations.lock`
- ZONE:package-json — `package.json`, `package-lock.json`, `yarn.lock`. Lock name: `package-manifest.lock`
- ZONE:env-config — `.env`, `config/environments/`. Lock name: `env-config.lock`
- ZONE:shared-test-fixtures — `test/fixtures/`. Lock name: `test-fixtures.lock`
```

The lock mechanism is intentionally simple: a file in a `tmp/locks/` directory that agents create and delete. For teams using Codex CLI's `PostToolUse` hooks, the lock deletion can be automated — a hook that fires on session exit removes any lock files held by the current session ID, preventing orphaned locks when a session terminates unexpectedly.

Exclusion zones address the failure mode that Yegge's survey respondents described most consistently: two agents modifying the same migration file, or two agents resolving the same dependency conflict in incompatible ways, without either agent knowing the other existed.[^1] Neither agent is wrong. The coordination layer is missing.

## Rollback Conditions

Rollback conditions define the state an agent should restore if it cannot complete its task, and the conditions under which it should attempt rollback rather than simply failing.

```markdown
## Rollback Conditions

If an agent encounters any of the following conditions, it must stop, log the condition
to `tmp/agent-log.md`, and execute the rollback procedure before exiting.

| Condition | Rollback Procedure |
|---|---|
| Test suite failure rate exceeds 5% above baseline | `git stash` all uncommitted changes; delete lock files; log session ID and condition |
| Schema migration fails mid-apply | Run `db:migrate:rollback`; restore `db/schema.rb` from HEAD; release `db-migrations.lock` |
| Dependency resolution produces version conflict in `package-lock.json` | `git checkout HEAD -- package-lock.json`; release `package-manifest.lock`; log conflicting dependency names |
| External API returns 5xx for more than 3 consecutive calls | Do not commit any changes that depended on the API response; release all held locks; log endpoint and response codes |

An agent must not proceed past its current task boundary when a rollback condition is triggered.
```

Rollback conditions serve a different function from exclusion zones. An exclusion zone prevents concurrent modification. A rollback condition handles the case where a single agent's work has destabilised shared state and should not be allowed to propagate further. The combination of the two means that agents coordinate before touching shared resources, and clean up after themselves when something goes wrong.

## Assembling the Manifest

The four additions fit naturally into a structured section of an existing AGENTS.md file. A team that already has a working AGENTS.md — tool definitions, hooks, model profiles — adds a `## Coordination` block at the top of the file, above the per-session instructions, so that agents read it first.

```markdown
# AGENTS.md — Repository: checkout-service

## Coordination
[Task Dependencies section]
[Milestones section]
[Exclusion Zones section]
[Rollback Conditions section]

## Instructions
[Existing per-session instructions]

## Tools
[Existing tool definitions]
```

Placing coordination above instructions is deliberate. An agent that reads a dependency annotation blocking its task should not need to parse the instruction section to know it should exit. Sequencing ensures that agents exit fast when blocked, rather than loading their full instruction context before discovering they cannot proceed.

## The Four-Agent Threshold

Teams running one or two agents in parallel can often manage coordination through human oversight — reviewing terminal tabs, checking git log, making the calls that prevent conflicts. The pattern breaks at four agents.[^4] At that point, the number of pairwise interactions between agents exceeds what a single developer can track in real time, and the expected value of "just check what the other agents are doing" drops below the cost of checking.

Four agents interacting pairwise create six interaction channels. Six agents create fifteen. The coordination overhead grows quadratically while the human oversight capacity stays flat. AGENTS.md as orchestration manifest is the point at which that overhead gets pushed into the repository, where it is persistent, readable by every agent, and reviewable in version history.

None of this requires a dedicated orchestration framework. The conventions described here are plain Markdown and plain files. They work with any agent that reads AGENTS.md on startup — which is every agent in the Codex CLI ecosystem by design.[^2] The cost of adoption is the time to write the coordination section. The cost of not adopting it, at four or more parallel agents, is the class of failures that Yegge's six hundred engineers were describing: correct agents, wrong coordination, lost work.

---

## Summary

AGENTS.md extends naturally from per-session instruction set to multi-agent orchestration manifest by adding four structured sections: dependency annotations (which tasks must complete before others begin), milestone tags (shared binary checkpoints agents read to confirm upstream state), exclusion zones (serialised resources with file-based locks agents acquire before modifying shared files), and rollback conditions (state-restoration procedures agents execute when they destabilise shared state).[^2] The pattern holds without additional tooling; it requires only that every agent in the team's session profiles reads AGENTS.md on startup, which is the default Codex CLI behaviour. Teams running fewer than four parallel agents can manage coordination through human oversight. At four and above, the pairwise interaction count exceeds practical human tracking capacity, and the conventions described here move that coordination cost into the repository where it is visible, versionable, and available to every agent in the session.[^4]

---

## Citations

[^1]: Yegge, S. (2026, September). *Thread: "Which IDE do you use to manage multiple coding agents?"* X (formerly Twitter). Reported in: codex-resources articles/2026-09-16-no-ide-for-agentic-ai-multi-agent-context-management-codex-cli.md

[^2]: OpenAI. (2026). *AGENTS.md specification and repository instruction format.* Codex CLI documentation. <https://github.com/openai/codex/blob/main/docs/agents-md.md>

[^3]: Jadhav, A., LaPlaca, M., Stone, R., Raja, S., Ochoa, L., & Nagaraju, P. (2026). *Compound Prompt Constraints in LLM Code Generation: A Factorial Study of Format, Persona, and Urgency.* arXiv:2609.03156.

[^4]: Herdr Engineering Blog. (2026, August). *Scaling Codex CLI: What breaks at four agents and how we fixed it.* Internal case study summarised in Yegge (2026) thread discussion.
