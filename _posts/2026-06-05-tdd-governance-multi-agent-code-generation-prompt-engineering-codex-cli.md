---
title: "TDD Governance for Multi-Agent Code Generation — Phase Gating, Bounded Repair, and Prompt-Level Enforcement for Codex CLI"
description: "A new framework from the University of Jyväskylä treats TDD not as a suggestion but as an enforceable control architecture for multi-agent coding systems. Here's what it means for Codex CLI workflows."
type: Technical Article
timestamp: 2026-06-05T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-06-05-tdd-governance-multi-agent-code-generation-prompt-engineering-codex-cli"
tags: ["tdd", "multi-agent", "prompt-engineering", "governance", "code-generation", "codex-cli", "testing", "agent-architecture"]
date: 2026-06-05T09:00:00+00:00
last_modified_at: 2026-09-03T20:11:01+01:00
---
# TDD Governance for Multi-Agent Code Generation — Phase Gating, Bounded Repair, and Prompt-Level Enforcement for Codex CLI


---

## The Problem with Procedural TDD in Agent Systems

We already know that telling a coding agent "always run tests before committing" can backfire. The TDAD paper (arXiv:2603.17973) demonstrated that generic TDD instructions increased regressions by 63% because the agent lacked structural knowledge about which tests to run.[^ch01-01] That paper solved the problem with dependency graphs — give the agent a map, not a lecture.

But dependency graphs address single-agent workflows. What happens when multiple agents collaborate on code generation — a planner, a test writer, an implementer, and a repair agent — and small logic errors propagate across the entire pipeline?

A new paper from the Universities of Jyväskylä and Tampere tackles exactly this: TDD governance as a distributed control architecture for multi-agent code generation.[^ch01-02]

## From Suggestion to Enforcement

The key insight is architectural. Rather than treating TDD as a development practice that agents should follow, the framework encodes TDD principles as enforceable constraints at the prompt and workflow level. The authors extracted three categories of constraint from canonical TDD sources (Beck, Martin):

**Order constraints** — test-first sequencing, Red-Green-Refactor phases. No implementation code exists until a failing test does.

**Granularity constraints** — minimal failing tests, minimal passing code. Agents cannot generate sweeping changes; each iteration is deliberately small.

**Feedback-quality constraints** — fast, deterministic, self-validating tests. No flaky tests, no manual inspection gates.[^ch01-03]

These constraints are formalised into a machine-readable JSON manifesto, then translated into prompt-level enforcement for each agent role.

## The Architecture: Separation of Proposal and Authority

The framework's most important design decision separates **proposal generation** from **state authority**. Language models generate structured patch proposals but cannot directly modify the workspace. A deterministic engine — not the LLM — decides whether a patch is applied.

This mirrors a pattern Codex CLI users will recognise: the agent proposes changes, but the sandbox controls execution. The paper extends this to multi-agent settings with four distinct roles:

### The Four-Agent Pipeline

1. **Planner** — Decomposes the task into test-sized increments. Its prompt encodes order constraints: each increment must start with a test specification, not an implementation sketch.

2. **Test Generator** — Writes minimal failing tests for the current increment. Its prompt encodes granularity constraints: one behaviour per test, no implementation logic, no mocking of unwritten code.

3. **Implementer** — Writes the minimum code to pass the current failing test. Cannot see test source code — only execution results and error logs. This enforced isolation prevents the agent from gaming the tests.

4. **Repair Agent** — Triggered when tests fail after implementation. Operates within bounded repair loops (maximum 3 iterations) with explicit termination conditions. If repair fails after 3 attempts, the increment is rolled back rather than forced through.[^ch01-04]

## Phase Gating: The Critical Mechanism

The framework's structural innovation is **phase gating**. Between each agent handoff, a validation gate applies three types of check:

**Structural checks** — Does the proposed change conform to the expected format? Is it a test (not implementation) during the test-writing phase?

**Policy checks** — Does the change respect TDD constraints? Has a failing test been written before any implementation code?

**Consistency checks** — Does the change align with the planner's decomposition? Has the scope drifted?

If any gate fails, the pipeline rejects the proposal and requests a revision — the LLM never bypasses the gate. This is deterministic enforcement, not probabilistic compliance.[^ch01-05]

## What This Means for Codex CLI

### Multi-Agent v2 Context

Codex CLI's multi-agent v2 runtime (shipped in v0.137.0) supports persistent agent sessions with follow-up defaults.[^ch01-06] As users build more sophisticated multi-agent pipelines — particularly with the OpenAI Agents SDK — the governance patterns from this paper become directly applicable.

### AGENTS.md Implications

The paper's findings reinforce what the TDAD research established: process lectures in AGENTS.md are counterproductive. Instead, effective TDD governance requires:

- **Structural constraints** embedded in the agent's context, not procedural instructions
- **Phase separation** that prevents agents from skipping the test-first step
- **Bounded iteration** with explicit rollback rather than infinite retry loops
- **Isolation** between test-writing and implementation contexts

For a practical Codex CLI setup, this translates to:

```markdown
## Testing Protocol

- Tests live in `tests/` and are NEVER modified by implementation agents
- Implementation agents receive test output only, not test source
- Maximum 3 fix attempts per failing test before rollback
- No implementation file may be created without a corresponding test file
```

### The Repair Loop Lesson

The bounded repair loop (maximum 3 iterations) is particularly relevant. Codex CLI users running agent pipelines often encounter infinite fix-retry cycles where the agent keeps attempting different approaches to a failing test. The paper's approach — hard termination with rollback after 3 attempts — prevents resource waste and context window exhaustion.

This aligns with practical experience: if an agent cannot fix a test failure in 3 attempts, the problem is usually in the test specification or the task decomposition, not in the implementation strategy.

## Comparison with TDAD

The two approaches are complementary rather than competing:

| Dimension | TDAD | TDD Governance |
|---|---|---|
| **Scope** | Single agent | Multi-agent pipeline |
| **Mechanism** | Dependency graph as agent skill | Prompt-level constraint enforcement |
| **Key insight** | Tell agents *which* tests, not *how* to test | Separate proposal from authority |
| **Regression reduction** | 70% (measured) | Preliminary (reduced retry cycles) |
| **Codex CLI fit** | AGENTS.md + pre-commit hooks | Multi-agent v2 pipeline orchestration |

A production-grade Codex CLI workflow could combine both: TDAD's dependency graph for structural knowledge within each agent, and TDD governance's phase gating for pipeline-level discipline.

## The Broader Pattern: Governance as Architecture

The paper's deepest contribution is not the specific TDD rules but the architectural pattern: **governance through structure, not instruction**. In the same way that a type system prevents entire categories of error without requiring developer discipline, prompt-level phase gating prevents entire categories of agent misbehaviour without requiring the LLM to "understand" TDD.

This pattern extends beyond testing. Configuration governance, security review gates, compliance checks — any discipline that developers occasionally skip under pressure can be encoded as a structural constraint rather than a procedural instruction.

For Codex CLI users building enterprise workflows, this is the direction: stop telling agents what to do, and start building architectures that make the wrong thing impossible.

---

## References

[^ch01-01]: Alonso, P., Yovine, S., and Braberman, V., "TDAD: Test-Driven Agentic Development — Reducing Code Regressions in AI Coding Agents via Graph-Based Impact Analysis," arXiv:2603.17973v2, March 2026. https://arxiv.org/abs/2603.17973

[^ch01-02]: Hasanli, T., Siddeeq, S., Khanal, B., Kotilainen, P., Mikkonen, T., and Abrahamsson, P., "TDD Governance for Multi-Agent Code Generation via Prompt Engineering," arXiv:2604.26615v1, April 2026. https://arxiv.org/abs/2604.26615

[^ch01-03]: Beck, K., *Test-Driven Development: By Example*, Addison-Wesley, 2003; Martin, R. C., *Clean Code*, Prentice Hall, 2008.

[^ch01-04]: Hasanli et al., Section 4: "The system enforces phase ordering, bounded repair loops, validation gates" through deterministic engine authority.

[^ch01-05]: Ibid., Section 5: Validation gates apply structural, policy, and consistency checks before mutations.

[^ch01-06]: OpenAI, "Codex CLI v0.137.0 Release Notes," June 2026. https://github.com/openai/codex/releases/tag/rust-v0.137.0
