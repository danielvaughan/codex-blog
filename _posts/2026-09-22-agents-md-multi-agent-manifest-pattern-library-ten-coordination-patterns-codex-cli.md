---
title: "AGENTS.md Multi-Agent Manifest Pattern Library: Ten Coordination Patterns for Parallel Agent Teams, Codex CLI"
parent: "Articles"
nav_order: 1165
date: 2026-09-22T09:00:00+00:00
last_modified_at: 2026-09-24T10:11:38+01:00
tags: ["codex-cli", "multi-agent", "agents-md", "orchestration", "coordination", "patterns", "agentic", "workflow"]
---

# AGENTS.md Multi-Agent Manifest Pattern Library: Ten Coordination Patterns for Parallel Agent Teams, Codex CLI


---

The AGENTS.md as Multi-Agent Orchestration Manifest article introduced a four-section coordination block that turns a per-session instruction file into a shared substrate for parallel agent teams: dependency annotations, milestone tags, exclusion zones, and rollback conditions.[^1] The architecture is sound, but architecture without implementation guidance is a specification, not a practice.

This article is the implementation layer. It presents ten copy-paste AGENTS.md coordination manifests for real-world multi-agent scenarios. Each pattern includes the full AGENTS.md block, the specific failure mode it prevents, the rollback condition it handles, and notes on where the four-agent threshold — the point at which pairwise interactions exceed practical human oversight — becomes relevant.

The patterns are ordered from simplest to most complex. Teams new to multi-agent coordination should start with Pattern 1 and add complexity incrementally as the workload demands it.

---

## The Coordination Block Template

Every pattern in this library uses the same section structure inside AGENTS.md:

```markdown
## Coordination

### Dependency Annotations
| task_slug | owned_by | status | depends_on | blocks |
|---|---|---|---|---|
| task-a | agent-1 | pending | — | task-b, task-c |
| task-b | agent-2 | pending | task-a | task-d |

### Milestone Tags
- `SCHEMA_MIGRATED`: Set by agent-1 after migration verified. Read by agents-2 and -3 before any ORM write.
- `API_CONTRACTS_FROZEN`: Set by agent-2 after OpenAPI spec committed. Read by front-end agent before component generation.

### Exclusion Zones
- `db/migrations/` — serialised. Agent owning task-a holds lock. All others read-only until `SCHEMA_MIGRATED`.
- `config/secrets.yml` — read-only for all agents. Human writes only.

### Rollback Conditions
| condition | owned_by | recovery |
|---|---|---|
| migration fails on first apply | agent-1 | `git stash pop`, restore schema snapshot from `tmp/schema-snapshot.sql` |
| API spec breaks existing consumers | agent-2 | revert to last `API_CONTRACTS_FROZEN` commit |
```

Agents read this block on session start. The dependency table tells each agent what it must wait for; milestone tags are boolean checkpoints written to `tmp/milestones/` as empty sentinel files; exclusion zones are enforced via `tmp/locks/`; rollback conditions give each agent an unambiguous recovery path without requiring human intervention for non-critical failures.[^2]

---

## Pattern 1: Database Migration + API Update Sequencing

**Scenario:** Two agents. Agent-1 runs the schema migration; agent-2 updates the ORM models and REST endpoints. The failure mode without coordination: agent-2 writes ORM code against the old schema while agent-1's migration is mid-flight, producing models that do not match the live database.

```markdown
## Coordination

### Dependency Annotations
| task_slug | owned_by | status | depends_on | blocks |
|---|---|---|---|---|
| run-db-migration | agent-1 | pending | — | update-orm-api |
| update-orm-api | agent-2 | pending | run-db-migration | — |

### Milestone Tags
- `SCHEMA_MIGRATED`: Written to `tmp/milestones/SCHEMA_MIGRATED` by agent-1 after `rails db:migrate` exits 0 and `rails db:schema:dump` completes. Agent-2 polls for this file before writing any model code.

### Exclusion Zones
- `db/schema.rb` — write-locked to agent-1 until `SCHEMA_MIGRATED`.
- `db/migrations/` — write-locked to agent-1 for the session duration.

### Rollback Conditions
| condition | owned_by | recovery |
|---|---|---|
| `rails db:migrate` fails | agent-1 | Run `rails db:rollback STEP=1`; write `tmp/milestones/SCHEMA_MIGRATION_FAILED`; stop |
| ORM model fails test suite | agent-2 | Revert ORM files; do not touch migration files |
```

**Four-agent threshold note:** This is a two-agent pattern. It is the baseline for all sequenced writes. Add agents only when ORM, REST, and GraphQL layers need parallel updates — at that point move to Pattern 2.

---

## Pattern 2: Front-End + Back-End + Test Agent Triad

**Scenario:** Three agents working in parallel: API implementation, UI component generation, and integration test authoring. The failure mode: test agent writes assertions against interface contracts that neither the API nor UI agent has finalised.

```markdown
## Coordination

### Dependency Annotations
| task_slug | owned_by | status | depends_on | blocks |
|---|---|---|---|---|
| implement-api | agent-1 | pending | — | author-tests |
| generate-ui | agent-2 | pending | — | author-tests |
| author-tests | agent-3 | pending | implement-api, generate-ui | — |

### Milestone Tags
- `API_CONTRACTS_FROZEN`: Written by agent-1 after OpenAPI spec committed to `api/openapi.yaml`.
- `UI_COMPONENTS_FROZEN`: Written by agent-2 after Storybook stories pass smoke tests.
- Agent-3 waits for both before writing any `describe` blocks.

### Exclusion Zones
- `api/openapi.yaml` — write-locked to agent-1.
- `src/components/` — write-locked to agent-2.
- `tests/integration/` — write-locked to agent-3 until both milestone tags present.

### Rollback Conditions
| condition | owned_by | recovery |
|---|---|---|
| OpenAPI spec breaks existing consumers | agent-1 | Revert `api/openapi.yaml`; clear `API_CONTRACTS_FROZEN` |
| UI Storybook smoke failures persist > 3 attempts | agent-2 | Write `tmp/milestones/UI_BLOCKED`; await human review |
```

---

## Pattern 3: Parallel Feature Branches with Shared Config

**Scenario:** Two or more agents developing independent features that all modify `config/feature-flags.yml`. The failure mode: merge conflicts in a file that appears trivially simple but causes silent feature-flag mismatches when overwritten.

```markdown
## Coordination

### Exclusion Zones
- `config/feature-flags.yml` — serialised. One agent appends its flag block, commits, then yields. Second agent pulls before appending. Never concurrent.

### Rollback Conditions
| condition | owned_by | recovery |
|---|---|---|
| Feature flag conflict detected on push | any | Pull, resolve with `--ours` for own flag only, re-push |
```

This minimal manifest is intentional. Shared-config patterns need exclusion, not full dependency graphs. Over-specifying the manifest is its own failure mode.[^3]

---

## Pattern 4: Overnight Rewrite with Test-Harness-First Gate

**Scenario:** A single long-running agent rewrites a service overnight. The harness-first requirement means the agent must pass the test harness before touching implementation files. The failure mode without the gate: agent implements before the harness exists and defines its own success criteria — producing code that satisfies self-authored tests rather than production-equivalent ones.

```markdown
## Coordination

### Dependency Annotations
| task_slug | owned_by | status | depends_on | blocks |
|---|---|---|---|---|
| build-test-harness | agent-1 | pending | — | rewrite-service |
| rewrite-service | agent-1 | pending | build-test-harness | — |

### Milestone Tags
- `HARNESS_PASS`: Written only after all golden-file tests pass against the *existing* implementation. This is the non-negotiable precondition. The agent must not begin rewriting until this file exists.

### Rollback Conditions
| condition | owned_by | recovery |
|---|---|---|
| Harness environment gap detected (queue/config mismatch) | agent-1 | Write `tmp/milestones/HARNESS_GAP_DETECTED`; stop; await human review |
| Rewrite fails harness after 5 iterations | agent-1 | `git stash`; write `tmp/milestones/REWRITE_BLOCKED`; preserve harness |
```

**Why this matters:** The Checkly overnight rewrite case study identified that a harness validated against three queues but production ran eighteen queues per region.[^4] The `HARNESS_GAP_DETECTED` sentinel gives the agent a sanctioned exit path rather than silently continuing with a defective harness.

---

## Pattern 5: Documentation + Code Agent Pair

**Scenario:** One agent updates implementation code; a second agent updates the documentation that references it. The failure mode: documentation agent reads stale source before the implementation agent has stabilised the public API surface.

```markdown
## Coordination

### Dependency Annotations
| task_slug | owned_by | status | depends_on | blocks |
|---|---|---|---|---|
| update-implementation | agent-1 | pending | — | update-docs |
| update-docs | agent-2 | pending | update-implementation | — |

### Milestone Tags
- `PUBLIC_API_FROZEN`: Written by agent-1 after all exported function signatures are committed and unit tests pass.

### Exclusion Zones
- `src/` — write-locked to agent-1 until `PUBLIC_API_FROZEN`.
- `docs/` — write-locked to agent-2; agent-1 reads only.
```

---

## Pattern 6: Security Scan Agent as Pre-Flight Gate

**Scenario:** A security-scan agent runs dependency audits and SAST checks before any deployment agent is allowed to proceed. The failure mode without the gate: a deployment agent pushes a build containing a known vulnerability because the scan was skipped or ran in parallel.

```markdown
## Coordination

### Dependency Annotations
| task_slug | owned_by | status | depends_on | blocks |
|---|---|---|---|---|
| security-scan | agent-1 | pending | — | build-deploy |
| build-deploy | agent-2 | pending | security-scan | — |

### Milestone Tags
- `SECURITY_GATE_PASS`: Written only when `npm audit --audit-level=high` and SAST tool both exit 0.

### Rollback Conditions
| condition | owned_by | recovery |
|---|---|---|
| High severity vulnerability found | agent-1 | Write `tmp/milestones/SECURITY_BLOCKED`; do not write `SECURITY_GATE_PASS`; write finding to `tmp/security-report.txt` |
```

---

## Pattern 7: Multi-Cloud Deployment Agents

**Scenario:** Parallel agents deploying to AWS and GCP. Both share a Terraform state backend. The failure mode: concurrent state writes corrupt the remote lock.

```markdown
## Coordination

### Exclusion Zones
- `terraform/` — serialised via `tmp/locks/terraform.lock`. Agent acquiring the lock runs `terraform apply`; releases lock on exit regardless of outcome.
- `terraform/backend.tf` — read-only for all agents; human writes only.

### Rollback Conditions
| condition | owned_by | recovery |
|---|---|---|
| `terraform apply` exits non-zero | holding agent | Run `terraform destroy -target=<changed_resource>`; release lock; write failure to `tmp/terraform-failure.log` |
| Lock file not released after 30 min | any | Write `tmp/milestones/TERRAFORM_LOCK_STALE`; await human review |
```

---

## Pattern 8: Sprint Batch with Decision-Impact Triage (4+ Agents)

**Scenario:** Four or more agents processing a sprint backlog simultaneously. Decision-impact scoring from the backlog's metadata drives task priority. The failure mode at this scale: pairwise interaction count reaches six channels for four agents and fifteen for six agents — human oversight can no longer track all combinations.[^2]

```markdown
## Coordination

### Dependency Annotations
| task_slug | owned_by | impact | status | depends_on | blocks |
|---|---|---|---|---|---|
| auth-refactor | agent-1 | HIGH | pending | — | payment-service, user-profile |
| payment-service | agent-2 | HIGH | pending | auth-refactor | — |
| user-profile | agent-3 | MEDIUM | pending | auth-refactor | — |
| docs-update | agent-4 | LOW | pending | user-profile | — |

### Milestone Tags
- `AUTH_CONTRACTS_FROZEN`: Written by agent-1. All HIGH-impact dependents wait for this before writing any auth-touching code.

### Exclusion Zones
- `src/auth/` — write-locked to agent-1 until `AUTH_CONTRACTS_FROZEN`.
- `tests/auth/` — write-locked to agent-1 for session duration.

### Rollback Conditions
| condition | owned_by | recovery |
|---|---|---|
| Auth refactor breaks downstream integration tests | agent-1 | Revert `src/auth/`; clear milestone; notify via `tmp/milestones/AUTH_BLOCKED` |
```

**Four-agent threshold note:** At four agents with cross-dependencies, this manifest is the minimum viable coordination layer. Without it, agents collide on auth code and waste turns resolving merge conflicts that could have been prevented by a five-line dependency table.

---

## Pattern 9: Long-Running Data Pipeline with Rollback Conditions

**Scenario:** A data-pipeline rewrite agent runs for multiple hours, processing large datasets. The failure mode: a partial write leaves the pipeline in an inconsistent state that is harder to recover from than a clean rollback.

```markdown
## Coordination

### Milestone Tags
- `BATCH_N_COMPLETE`: Written after each pipeline batch completes validation. Allows resumption from the last clean checkpoint rather than full restart.

### Rollback Conditions
| condition | owned_by | recovery |
|---|---|---|
| Batch validation fails | agent-1 | Do not advance batch counter; write failure details to `tmp/pipeline-failures/batch-N.log`; retry up to 3 times before writing `tmp/milestones/PIPELINE_BLOCKED` |
| Schema drift detected mid-run | agent-1 | Write `tmp/milestones/SCHEMA_DRIFT`; restore last `BATCH_N_COMPLETE` snapshot from `tmp/checkpoints/`; stop |
| Disk / memory limit approached | agent-1 | Flush current batch; write checkpoint; write `tmp/milestones/RESOURCE_LIMIT`; stop cleanly |
```

The checkpoint-per-batch pattern comes directly from the ACID-compliant agent transaction model — atomicity at the batch level, durability via the append-only checkpoint log.[^5]

---

## Pattern 10: Review Agent as Finaliser

**Scenario:** All implementation agents have completed. A dedicated review agent reads every changed file, runs the test suite, checks citation or link validity, and produces a sign-off report. The failure mode: no agent owns the cross-cutting quality check, so it is either skipped or duplicated inconsistently across agents.

```markdown
## Coordination

### Dependency Annotations
| task_slug | owned_by | status | depends_on | blocks |
|---|---|---|---|---|
| implement-feature-a | agent-1 | pending | — | final-review |
| implement-feature-b | agent-2 | pending | — | final-review |
| final-review | agent-3 | pending | implement-feature-a, implement-feature-b | — |

### Milestone Tags
- `REVIEW_COMPLETE`: Written by agent-3 only after: (1) all tests pass, (2) no broken links detected, (3) review report written to `tmp/review-report.md`.

### Exclusion Zones
- All `src/` and `tests/` directories — read-only for agent-3. Review agent does not modify implementation files.

### Rollback Conditions
| condition | owned_by | recovery |
|---|---|---|
| Test suite fails in review | agent-3 | Write failures to `tmp/review-failures.md`; do not write `REVIEW_COMPLETE`; implementation agents triage their own failures |
```

---

## Choosing the Right Pattern

Not every scenario needs all four coordination block sections. The following guides selection:

| Primary risk | Section needed |
|---|---|
| Sequential dependency violation | Dependency Annotations + Milestone Tags |
| Shared-file collision | Exclusion Zones only |
| Partial-write inconsistency | Rollback Conditions only |
| Four or more concurrent agents | Full coordination block |

A manifest that is larger than the risk it addresses adds overhead without safety. Lulla et al. found that AGENTS.md files reduce agent runtime by 28.64 per cent and output token consumption by 16.58 per cent while maintaining comparable task completion — but only when the instructions are precise and well-targeted.[^3] Oversized manifests reduce that gain.

Start with the smallest pattern that addresses the specific failure mode. Add sections when a new failure mode appears, not before.

---

## Footnotes

[^1]: Vaughan, D. (2026, September 21). *AGENTS.md as multi-agent orchestration manifest: Dependency annotations, exclusion zones, codex CLI*. codex-resources. https://danielvaughan.github.io/codex-resources/articles/2026-09-21-agents-md-multi-agent-orchestration-manifest-dependency-annotations-exclusion-zones-codex-cli
[^2]: Herdr Engineering Blog. (2026, August). *The four-agent threshold: Why pairwise interaction count is the practical limit for unstructured multi-agent coordination*. herdr.io/engineering. Retrieved 2026-09-22.
[^3]: Lulla, J. L., Mohsenimofidi, S., Galster, M., Zhang, J. M., Baltes, S., & Treude, C. (2026). *On the Impact of AGENTS.md Files on the Efficiency of AI Coding Agents*. arXiv:2601.20404. https://arxiv.org/abs/2601.20404
[^4]: Janusevicius, E. (2026, September). *We let AI agents rewrite a 92M-message-a-day service in Go. Zero incidents*. Checkly Engineering Blog. https://www.checklyhq.com/blog/agentic-rewrite-nodejs-to-go/
[^5]: Sun, Z., Wang, X., & Li, G. (2026). *Agentic transaction: Towards ACID-compliant agent systems*. arXiv:2608.13900. https://arxiv.org/abs/2608.13900
