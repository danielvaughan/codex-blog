---
title: "The Agentic Engineering Pod: Three Roles, One Shared Context Layer"
date: 2026-03-26
tags: agentic-pod, roles, context-architect, value-engineer, quality-engineer, three-person-rule, codex-cli
---

![Sketchnote](/sketchnotes/articles/2026-03-26-agentic-pod-roles-and-codex.png)

---

## The Problem

Traditional software team structures were not designed for agentic delivery. They are built around specialisations: product managers own requirements, engineers write code, QA runs tests, DevOps manages pipelines. When agents can do most of that execution, you are not solving a staffing problem — you are running the wrong organisational model. Adding AI tools to an existing hierarchy is like bolting a jet engine onto a bicycle. The bicycle frame was not designed for that kind of force, and neither were the coordination patterns built around it.

The question in an agentic team is not who writes the code. It is who owns the context the agents run inside, who orchestrates the delivery, and who verifies that what the agents produced can be trusted in production. Those are three distinct problems, and they require three distinct roles.

## The Pod Model

The Agentic Engineering Pod has three roles. **All three are human.** Each person owns a distinct question; the agents are tools they wield, not roles in the pod.

The **Context Architect** *(human)* owns the *Why* and the *What*: business outcomes, product context, architectural standards, guardrails, and the rules under which agents operate. Their primary artefact is not a feature — it is the AGENTS.md hierarchy and the acceptance criteria framework that every agent session runs inside.

The **Value Engineer** *(human)* owns the *How*: delivery orchestration, agent supervision, and the expert judgment that catches what agents miss. They are the primary Codex user in the pod. They write briefs, run sessions, review output, and provide the engineering depth to redirect agents when output drifts from intent.

The **Quality Engineer** *(human)* owns the *Trust*: the CI/CD pipelines, automated gates, security controls, and the quality veto. They engineer the platform that determines what the pod can safely ship. Their veto authority is structural — it cannot be overridden by the Context Architect or the Value Engineer.

Three roles, three layers. The pod works because each layer depends on the others: the Value Engineer's orchestration is only as good as the context the Context Architect provides; the Quality Engineer's gates only mean something if the Value Engineer's sessions produce verifiable output. Remove any one role and the system degrades in a predictable, characteristic way.

## Context Architect — Human Role

The Context Architect translates business intent into a delivery system agents can execute. They are accountable for the pod's effectiveness — not feature-by-feature, but at the level of whether the context layer keeps agents aligned with outcomes over time.

In Codex, the Context Architect's primary work is the AGENTS.md hierarchy. Repo-level, subdirectory-level, role-specific — each layer of the hierarchy encodes the context that shapes every session the Value Engineer runs and every gate the Quality Engineer fires. A well-authored AGENTS.md specifies product outcomes, acceptance criteria format, architectural constraints, approved dependencies, and the autonomy levels that apply to different categories of change.

**Concrete example.** A subscription service is adding a renewal flow. Before the feature enters delivery, the Context Architect assesses whether `src/subscriptions/AGENTS.md` covers the idempotency requirements for the renewal path. It does not. They run a context-architect session:

```
Update src/subscriptions/AGENTS.md to add acceptance criteria for the renewal
path. The endpoint must be idempotent (double-call safe), emit a
subscription.renewed event before returning, and leave state unchanged on
any Stripe failure. Flag the autonomy level: this is a write-path change.
```

The result: before a single line of implementation code is written, every agent that touches the renewal feature runs inside a context that knows exactly what "correct" means for this path.

The Context Architect also owns the **autonomy ladder**: the progression from *assist* (agent suggests, human applies) through *approval-required* and *policy-bounded* to *self-remediating* (agent acts within guardrails). Graduating between levels requires evidence — test history, audit logs, review records — not confidence.

## Value Engineer — Human Role

The Value Engineer is the engine of the pod. They own the delivery workflow from specification to shippable increment, and they are the primary driver of Codex sessions.

Their craft skill is **briefing engineering**: writing high-signal prompts that give agents enough intent, constraints, examples, and edge cases to execute without ambiguous decision points. A vague brief produces unpredictable output. A specific brief produces convergent output on the first attempt. The difference is not length — it is the presence of explicit failure scenarios, idempotency requirements, out-of-scope boundaries, and references to the relevant AGENTS.md context.

They also run the **agentic loop**: deciding when to let an agent run to completion, when to interrupt and redirect, and when a session has drifted so far it should be restarted. This requires deep engineering judgment — the ability to look at agent output and spot a race condition, an architectural inconsistency, or a correctness failure that the agent's generated tests will not catch.

**Concrete example.** The Value Engineer receives the renewal feature from the Context Architect with the acceptance criteria already written. They open a Codex session in a dedicated worktree and lead with the high-signal brief:

```
Implement POST /subscriptions/renew per the acceptance criteria in
src/subscriptions/AGENTS.md. Start with /plan — show me the affected files
and dependency order before writing any code. The idempotency check must use
a database-level unique constraint, not an application-level read-check-write.
```

The agent produces an implementation plan. The Value Engineer reviews it, approves execution, and watches the session — approving each write-path action per the Level 2 autonomy policy the Context Architect specified. When the agent's first idempotency approach has a race condition, the Value Engineer catches it and redirects. Expert judgment, not agent judgment.

## Quality Engineer — Human Role

The Quality Engineer does not just run tests. They engineer the trust platform: the CI/CD pipelines, automated gates, security controls, and the hooks that enforce verification at every point in the delivery workflow.

Their primary tools are **hooks** and **automated verification runs**. Post-session hooks fire when a Codex session ends, running the test suite, checking that no files outside the permitted scope were modified, verifying that audit events were emitted. Pre-prompt hooks scan incoming task descriptions for prompt injection patterns before they enter agent context. These hooks are the Quality Engineer's authorship — the verification layer that runs regardless of what the agent produced.

They also own **adversarial testing**: proactively red-teaming the pod's agent workflows for prompt injection, unsafe tool use, and data exposure. Not periodically. Continuously, as the agentic system's capabilities expand.

**Concrete example.** After the Value Engineer signals completion on the renewal implementation, the Quality Engineer's post-session hook fires automatically: test suite runs, scope check verifies no restricted files were touched, audit event verification confirms the `subscription.renewed` event is emitted. Then the Quality Engineer runs an adversarial session:

```
Red-team POST /subscriptions/renew. Test for concurrent calls from the same
user_id, Stripe webhook arrival during a pending renewal, and any path where
a second charge could be created for the same user. Report findings.
```

The database-level constraint holds. VERIFICATION_PASSED. The feature is merge-ready.

The Quality Engineer's **quality veto** is what makes this meaningful. If verification fails, the feature does not ship — regardless of who produced it or how urgently it is needed. The veto is structural, not political.

## The Three Principles

Three operating principles govern how the pod delivers:

**Fix Time/Budget, Flex Scope.** The pod protects deadlines by flexing scope, not by extending timelines. The 90/10 path — 90% of the outcome with 10% of the effort, using agentic automation on the routine work — is the default target. When a deadline is at risk, reduce scope. Deliver the core value and iterate on real-world usage.

**The Three-Person Rule.** Three people, no more. Every person in the pod is a producer; no one coordinates other people. People direct agents; they do not manage each other. When demand grows, launch a parallel pod that shares the platform layer — do not expand the existing pod. Two pods of three scale better than one pod of six.

**Sketch First, Code Last.** Validate the workflow before committing to production code. Diagrams, pseudocode, throwaway proof-of-concepts. Production code is expensive to change even when agents wrote it. The Value Engineer runs `/plan` and reviews it before approving execution. The Context Architect does not provide full acceptance criteria until the sketch is validated.

## The Pod Doesn't Just Make Delivery Faster

The pod makes delivery safer. Agents operating with good context, well-engineered briefs, and structural verification gates are safer than agents operating without those things — and safer than human teams that lack the automated verification density the Quality Engineer builds. The three roles are not overhead; they are the safety envelope that makes high-autonomy agent operation trustworthy.

Small teams with agents can outperform large teams without them — but only when the three layers are working. Get the context right. Get the briefs right. Get the gates right. The rest follows.

---

[^1]: AGENTS.md hierarchy, loading order, and override semantics: see Chapter 7 of *Engineering with Codex CLI*.

[^2]: Hooks configuration and enforcement — SessionStart, SessionStop, userpromptsubmit: see Chapter 12.

[^3]: Worktrees as parallel execution infrastructure for Value Engineer sessions: see Chapter 19.
