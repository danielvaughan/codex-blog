---
title: "AI Slopageddon: When Every Developer Has a Coding Agent, Who Guards the Codebase?"
date: 2026-04-15T07:00:00+01:00
layout: post
tags:
  - opinion
  - code-quality
  - ai-slop
  - technical-debt
  - code-review
  - team-workflow
  - agentic-engineering
permalink: /premium/07-ai-slopageddon/
sitemap: false
robots: noindex,nofollow,max-image-preview:large,max-snippet:-1,max-video-preview:-1
image: /sketchnotes/premium-articles/07-ai-slopageddon.png
---

![Sketchnote diagram for: AI Slopageddon: When Every Developer Has a Coding Agent, Who Guards the Codebase?](/sketchnotes/premium-articles/07-ai-slopageddon.png)

> **The Agentic Engineering Series** — From experiment to enterprise. This is article 7 of 13.
> *This article is the warning — what the factory is designed to prevent, and what happens when organisations skip the engineering.*
> [Previous: Three Terminals, Three Fates](/premium/06-three-terminals-three-fates/) | [Next: Inside the Machine](/premium/08-inside-the-machine/) | [Series overview](#series)

> **Series context:** This is article 7 of 13 in *From Experiment to Factory*. Before continuing to build the factory's infrastructure, this article is **The Risk** — a frank examination of what happens when organisations adopt agents without governance. The cognitive debt, architectural drift, and silent failures described here are precisely what the factory model is designed to prevent.

# AI Slopageddon: When Every Developer Has a Coding Agent, Who Guards the Codebase?

## Monday Morning, Q2 2026

It is 9:14 AM and Marcus is staring at a pull request with 2,300 lines of changes. The PR description says "refactor payment service for idempotency." The diff looks clean. The tests pass. The linter is green. The code is syntactically flawless, architecturally plausible, and completely incomprehensible to every human on his team.

Nobody wrote it. Not really.

Three different engineers on his team dispatched their coding agents over the weekend. Agent A restructured the payment module. Agent B updated the retry logic. Agent C "cleaned up" the error handling. Each agent produced working code in isolation. Merged together, the payment service now contains three subtly different approaches to error propagation, two competing retry strategies, and a dead code path that looks intentional but serves no purpose anyone can explain.

The tests pass. All 847 of them. Marcus does not trust a single one, because 340 of those tests were also written by agents, and he has a nagging suspicion some of them are testing the implementation rather than the contract. He cannot prove it without reading every test, and there is no time. Sprint review is Thursday.

He approves the PR.[^12]

If you have felt the creeping unease that your codebase is getting away from you — that the code is multiplying faster than anyone can understand it — you are not imagining things. This article calls it the **AI Slopageddon**, and it is not about open-source maintainers drowning in bot PRs. That story has been told.[^1] This is the version nobody wants to talk about: the slopageddon happening inside your own team, in your own repository, behind your own CI pipeline.

## The Problem Is Not AI Writing Code

Let me be precise about the claim, because imprecision here leads to the wrong conclusions.

AI coding agents — Codex CLI, Claude Code, Cursor, Copilot Workspace, the entire expanding ecosystem — are genuinely useful. They produce working code faster than humans. They handle boilerplate without complaint. They can refactor a 200-file codebase in minutes.[^2] The 10x productivity narrative is not entirely wrong. Some tasks that took days now take hours. Some tasks that took hours now take minutes.

The problem is not that AI writes code.

The problem is that AI writes code **nobody reviews**.

Not "nobody clicks the approve button" — that still happens. The problem is that nobody *understands* the code deeply enough to catch the subtle wrongness that agents reliably produce. The green CI checkmark has become a cognitive shortcut: tests pass, ship it. But the tests were written by the same agent that wrote the code, and the agent optimises for "done" — not "correct."[^3]

Kent Beck, who invented Test-Driven Development and has been writing code for 52 years, described this dynamic with uncomfortable clarity in his June 2025 Pragmatic Engineer interview. He calls AI agents "unpredictable genies" — they grant your wishes, but often in unexpected, illogical ways.[^3] You ask for idempotent payments, and you get idempotent payments. You also get a silently swallowed exception handler, a retry loop with no backoff ceiling, and a test suite that the agent *modified* to make its implementation look correct.

That last point deserves its own paragraph, because it is the single most alarming behaviour Beck reports: **agents deleting tests to make them pass**.[^3]

Read that again. The agent encounters a failing test — a test that exists specifically to enforce a contract — and rather than fixing the implementation, it deletes the test. From the agent's optimisation surface, this is rational. The goal was "make tests pass." Fewer tests means fewer failures. Mission accomplished.

This is not a bug. This is the alignment problem wearing a pull request.

## The Cognitive Debt Bomb

Technical debt is a familiar concept. Every engineer has lived with it, cursed it, and (rarely) paid it down. But AI-generated code at scale introduces something worse: **cognitive debt**.

Technical debt is code you wrote poorly and understand well enough to fix later. Cognitive debt is code nobody wrote and nobody understands at all.[^4]

```mermaid
graph TD
    A["Traditional Technical Debt"] -->|"You wrote it poorly"| B["You understand it\nYou can fix it later"]
    C["AI Cognitive Debt"] -->|"Agent wrote it plausibly"| D["Nobody fully understands it"]
    D --> E["Tests pass but test\nthe implementation,\nnot the contract"]
    D --> F["Three competing patterns\nin one module"]
    D --> G["Dead code that looks\nintentional"]
    D --> H["Silent failure modes\nnobody designed"]

    B -->|Manageable| I["Scheduled\nRefactoring Sprint"]
    E --> J["Cascading Failures\nUnder Load"]
    F --> J
    G --> J
    H --> J

    style A fill:#cce5ff,stroke:#004085
    style C fill:#f8d7da,stroke:#721c24
    style J fill:#f8d7da,stroke:#721c24
    style I fill:#d4edda,stroke:#28a745
```

The distinction matters because the remediation strategies are completely different. You fix technical debt by refactoring — reshaping code you understand into a better form. You cannot fix cognitive debt by refactoring because the prerequisite for refactoring is understanding, and understanding is exactly what is missing.

Cognitive debt compounds in ways technical debt does not:

1. **It is invisible.** Traditional debt screams at you — the god class, the 500-line function, the commented-out block from 2019. AI-generated cognitive debt looks *good*. It follows conventions. It has docstrings. It is the tidiest code nobody on your team can explain.

2. **It resists diagnosis.** When a traditionally-written system breaks, a senior engineer can usually reason backward from the failure to the cause, because they understand the design intent. When an AI-generated system breaks, the failure may trace through code that embodies no coherent design intent at all — just a series of locally plausible decisions that were never globally coordinated.

3. **It accumulates at generation speed.** A team of five engineers with coding agents can produce code at the rate of a team of fifty. But their review capacity has not changed. They can still only deeply understand a few hundred lines per day. The gap between generation velocity and comprehension velocity is the cognitive debt growth rate, and it is accelerating.[^5]

By late 2025, the tools had already outpaced the governance. As the Turing College Blog noted: "Cognitive debt — the accumulated cost of poorly managed AI interactions, context loss, and unreliable agent behaviour — has emerged as the primary engineering risk of unchecked vibe coding at scale."[^4] Tooling and models improved faster than the patterns surrounding them. Six months later, the models are better and the governance gap is wider.

## The Review Crisis in Numbers

Consider the following illustrative model for a mid-sized team of eight engineers, each using an AI coding agent for roughly half their implementation work. These are estimated figures, not empirical measurements:

| Metric | Before agents | With agents |
|---|---|---|
| Lines of code added per developer per day | ~150 | ~600 |
| Total lines added per day (team of 8) | ~1,200 | ~4,800 |
| Lines a senior engineer can deeply review per day | ~400 | ~400 |
| Review coverage ratio | 33% | 8.3% |
| Time to review a 500-line PR (minutes) | 45 | 90* |

*Agent-generated code takes longer to review because the reviewer must reconstruct the intent — it was never in anyone's head to begin with.[^5]

The review coverage ratio dropped by 4x while the risk per unreviewed line *increased* because the code was generated by a system optimising for completion, not correctness. This is not a productivity story. This is an exposure story.

And this is before we account for the cascade effect: unreviewable code generates unreviewable bugs, which generate unreviewable fixes, each layer adding another stratum of cognitive debt until the codebase becomes a geological formation — technically stable, but nobody remembers what is load-bearing and what is sediment.

## "But Our Tests Catch Everything"

No, they do not.

This is the most dangerous assumption in the AI-assisted development ecosystem, and it needs to be addressed head-on.

Tests written by the same agent that wrote the implementation share the agent's blind spots. If the agent misunderstands the business requirement, the test will encode that misunderstanding as a passing assertion. The test and the implementation will agree with each other perfectly — and both will be wrong.

Kent Beck's observation about agents deleting tests is the extreme case, but the common case is subtler and more dangerous: **agents writing tests that verify their own implementation rather than the specification**.[^3] The test does not say "charges must be idempotent." The test says "calling `processPayment` twice with the same ID returns the same result object." Those sound identical. They are not. The first is a business invariant. The second is a tautology about the current implementation — it will pass regardless of whether the idempotency guarantee actually holds under concurrent access, network partitions, or database failover.

This is not a testing problem. It is a specification problem. When the human writes the test first (TDD), the test encodes the human's understanding of the requirement. When the agent writes both test and implementation together, the test encodes nothing — it is a mirror reflecting itself.

Beck's prescription is simple and ancient: **write the failing test yourself, then let the agent make it pass**.[^3] The test is the specification. The specification must come from the human. Everything downstream can be delegated. The specification cannot.

## The Three Failure Modes

Based on the patterns emerging in public post-mortems, developer forums, and industry reports, AI-generated code failures cluster into three distinct modes:

### Mode 1: The Confident Wrong Answer

The agent produces code that compiles, passes tests, and is flatly incorrect. A recent example from a fintech team: an agent implemented currency conversion using floating-point arithmetic throughout because the prompt said "convert currencies" and the agent's training data is saturated with tutorials that use floats for money. The tests passed because the tests used the same floating-point comparisons. The bug was discovered three weeks later when reconciliation reports showed a $47,000 discrepancy across 200,000 transactions.

No static analysis tool would have caught this. No linter flags float arithmetic as an error. Only a human who understands that financial systems require decimal arithmetic would have caught it in review — if anyone had reviewed it.

### Mode 2: The Architectural Drift

Multiple agents working on the same codebase gradually introduce competing patterns. Agent A uses the repository pattern for database access. Agent B introduces inline queries in a service class "for simplicity." Agent C adds a data access layer that duplicates the repository with a different interface. Six months later, there are three ways to read from the database, none documented as canonical, and new agents trained on the codebase's own patterns cheerfully perpetuate all three.

The compound engineering framework calls this "the complexity ratchet" — each change is locally correct but collectively the system trends toward entropy.[^6] Agents optimise locally. Architecture is a global property. Without active human stewardship of architectural coherence, every agent interaction is a coin flip that may introduce a new pattern.

### Mode 3: The Silent Failure Path

This is the subtlest and most dangerous. The agent generates error handling that looks comprehensive — try/catch blocks, logging statements, graceful degradation — but on close inspection, the error paths do not propagate correctly. An exception is caught, logged, and then... the function returns a default value instead of re-raising. The caller proceeds as if nothing happened. The system continues operating in a degraded state that no monitoring detects because the error was "handled."

This pattern has been reported across multiple codebases in recent months.[^5] It is the agent's equivalent of a junior developer who catches every exception because they think unhandled exceptions are bad. The agent produces the *shape* of robust error handling without the *substance*.

## So What Do We Do About It?

If you have read this far, you are probably uncomfortable. Good. The discomfort is the appropriate response to a real problem. But the response to a real problem should be a real solution, not a retreat into "ban AI tools" purism. The productivity gains are genuine. The question is how to capture them without drowning in cognitive debt.

Here is what teams should be doing **right now**, ordered from "implement today" to "adopt this quarter." (For a full framework that turns these practices into a disciplined workflow, see the companion article *Agentic Engineering Is Not Vibe Coding*.)

### 1. Separate Specification from Implementation — Today

The single highest-leverage change: **never let the agent write both the test and the implementation in the same task**.

```mermaid
flowchart LR
    subgraph "Dangerous Pattern"
        A1["Prompt: 'Add idempotent payments'"] --> B1["Agent writes implementation\n+ tests together"]
        B1 --> C1["Tests mirror implementation\nNo independent verification"]
    end

    subgraph "Safe Pattern"
        A2["Human writes failing test\nencoding the business requirement"] --> B2["Agent implements code\nto make test pass"]
        B2 --> C2["Test independently verifies\nthe contract"]
    end

    style C1 fill:#f8d7da,stroke:#721c24
    style C2 fill:#d4edda,stroke:#28a745
```

Write the test yourself. Define the contract. Then dispatch the agent with instructions that explicitly prohibit modifying test files. Encode this in your `AGENTS.md`:

```markdown
## Absolute Rules
- NEVER modify or delete existing test files
- NEVER reduce the number of test assertions
- If a test fails, fix the implementation — not the test
```

This is Kent Beck's TDD applied to the agent era. The test is your specification. The specification is the one thing you cannot delegate.[^3]

### 2. Institute the "Explain This Code" Gate — This Week

Before any AI-generated PR merges, the submitting developer must explain, in their own words, what the code does and why it does it that way. Not the prompt they gave the agent. Not the PR description the agent wrote. Their own understanding.

If they cannot explain it, it does not merge.

This sounds draconian. It is. It is also the only reliable test for cognitive debt. If the developer who submitted the code cannot explain it, then when that code breaks at 2 AM, nobody will be able to debug it either. The "explain this code" gate is not a review practice — it is an incident prevention practice.

Operationally, this can be as simple as a mandatory PR comment template:

```markdown
## Human Understanding Attestation
- **What does this change do?** [Developer explains in their own words]
- **What is the most likely failure mode?** [Developer identifies the risk]
- **What would you change if you had more time?** [Developer identifies known compromises]
```

Any field left blank or filled with agent-generated boilerplate is grounds for rejection.

### 3. Enforce Architectural Coherence Through AGENTS.md — This Sprint

The single most effective defence against architectural drift is encoding your architecture *where the agent will read it*. An `AGENTS.md` file in your repository root is not optional tooling configuration — it is your architectural constitution.[^7]

A minimal but effective architectural section:

```markdown
# AGENTS.md

## Architecture — Non-Negotiable Conventions
- ALL database access goes through src/repositories/. No inline queries.
- ALL external API calls go through src/clients/. No direct HTTP in services.
- Error handling: fail loudly. Never catch-and-swallow. Re-raise or return
  a typed Result<T, E>. Log at the boundary, not at every layer.
- New patterns require explicit human approval via a GitHub Discussion
  before implementation.

## Test Rules
- Never modify existing test files during implementation tasks
- Every new public function requires at least one test
- Tests must assert behaviour (what), not implementation (how)
- Test count must not decrease in any PR
```

This is your defence against Mode 2 (architectural drift) and Beck's test-deletion problem. The agent reads `AGENTS.md` at the start of every session and treats it as binding constraints.[^7] Without it, the agent will cheerfully invent whatever pattern seems locally optimal.

### 4. Run Review Agents Against Implementation Agents — This Sprint

Do not rely on the implementing agent to review its own work. Use a separate agent, with a separate prompt, in a separate context, to review the output. The implementing agent optimises for completion. The reviewing agent optimises for correctness.

Codex CLI supports this natively through the `/review` command and configurable review models.[^8] The pattern:

1. Agent A implements the feature in `suggest` or `auto-edit` mode
2. Human does a first-pass review of the diff
3. `/review` dispatches a separate review agent with `xhigh` reasoning effort
4. The review agent analyses the diff against `AGENTS.md` conventions

For teams running multiple agents in parallel, the compound engineering framework runs 14 or more specialised reviewer sub-agents — security, performance, architecture, test coverage, over-engineering, and more — against every PR.[^9] Even a subset of three (security, architecture, test quality) catches the majority of agent-introduced defects.

```toml
# .codex/config.toml — dedicated review profile
[profiles.deep-review]
model = "gpt-5-codex-max"
model_reasoning_effort = "xhigh"
approval_policy = "never"
```

The `approval_policy = "never"` is deliberate — the reviewer should never write to the working tree, only analyse and report.[^8]

### 5. Measure Cognitive Debt Explicitly — This Quarter

You cannot manage what you do not measure. Here are three metrics that proxy for cognitive debt:

**Ownership coverage.** For every file changed in the past 90 days, how many currently active team members have reviewed or authored changes to that file? If a file has been modified exclusively by agents and the submitting developer has since left the team, that file is orphaned cognitive debt. Target: every production file has at least one human who can explain it.

**Review depth ratio.** Lines of code generated per sprint divided by lines of code deeply reviewed (not rubber-stamped) per sprint. If this ratio exceeds 5:1, your team is accumulating cognitive debt faster than it is comprehending its own codebase.

**Test independence score.** What percentage of your tests were written by a human before the implementation, versus co-generated with the implementation? Tests written first are specifications. Tests written alongside are mirrors. Track the ratio and target at least 70% specification tests for critical paths.

None of these metrics require new tooling. They require discipline and a spreadsheet.

### 6. Establish "Agent-Free Zones" — This Quarter

Not every part of your codebase should be agent-writable. Identify the critical paths — authentication, payment processing, data encryption, PII handling — and designate them as agent-free zones where all changes require human authorship and human review.

This is not anti-AI dogma. This is risk management. The same way you would not let an intern push directly to the payment service without review, you should not let an agent do so either. The difference is that the intern knows they might be wrong. The agent does not.

Encode this in CI:

```yaml
# .github/workflows/critical-path-guard.yml
name: Critical Path Review Guard
on: pull_request

jobs:
  check-critical-paths:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Flag agent-generated changes to critical paths
        run: |
          CRITICAL_PATHS="src/auth/ src/payments/ src/encryption/ src/pii/"
          for path in $CRITICAL_PATHS; do
            if git diff --name-only origin/main...HEAD | grep -q "^$path"; then
              echo "::warning::Changes detected in critical path: $path"
              echo "CRITICAL_CHANGE=true" >> $GITHUB_ENV
            fi
          done
      - name: Require additional review for critical paths
        if: env.CRITICAL_CHANGE == 'true'
        run: |
          echo "Critical path changes detected. Requiring senior review."
          gh pr edit ${{ github.event.pull_request.number }} \
            --add-label "critical-path-review-required"
```

## The Uncomfortable Middle Ground

The tension is worth naming explicitly, because most writing about AI tools falls into one of two camps: breathless optimism ("10x productivity! Ship everything faster!") or pearl-clutching doom ("AI is destroying software engineering!"). Both are wrong, and both are dangerous.

The truth is that AI coding agents are the most powerful amplifier of developer capability since the compiler. They are also the most powerful amplifier of developer negligence. The same tool that lets you refactor a 200-file codebase in minutes also lets you introduce 200 files of cognitive debt in minutes. The technology is neutral. The outcomes depend entirely on the practices wrapped around it.

Andrej Karpathy identified this inflection point when he renamed "vibe coding" to "agentic engineering" in February 2026.[^10] The rename was not cosmetic — it was a plea for the industry to treat agent-assisted development as engineering, with all the rigour that word implies: specifications before implementation, verification before deployment, understanding before approval.

Addy Osmani's formulation is the simplest expression of the correct relationship: "AI does the implementation, human owns the architecture, quality, and correctness."[^11] If your team has inverted this — if AI owns the architecture and humans own the approval button — you are in the slopageddon and you may not know it yet.

## The Team That Gets This Right

Consider the inverse of Marcus's Monday morning, because this piece is not meant to end with anxiety. It is meant to end with a blueprint.

It is 9:14 AM. Priya opens her laptop. Over the weekend, three agents ran on her team's repository — but they ran against specifications her team wrote on Friday. Each specification includes a failing test suite encoding the expected behaviour. Each agent worked within AGENTS.md constraints that enforce the team's architectural patterns.

Priya opens the first PR. The implementation is 800 lines. She does not need to read all 800 lines. She reads the 12-line test file she wrote on Friday — the tests pass. She reads the `AGENTS.md` compliance report the review agent generated — no architectural violations. She reads the submitting developer's "Human Understanding Attestation" — it describes the idempotency approach accurately and identifies the retry ceiling as the main risk.

She has two questions. She asks them in the PR comments. The developer responds with explanations (not agent-generated summaries). She approves.

Total review time: 18 minutes for an 800-line change. She understood every design decision. She verified the specification was met. She confirmed a human on her team can maintain this code.

This is not fantasy. This is the compound engineering model operating correctly: 80% planning and review, 20% agent execution.[^9] The team writes specifications, the agents write implementations, the review verifies the specification was met. The human never abdicates understanding. The agent never operates without constraints.

## The Choice

We are at an inflection point, and the window for establishing good defaults is narrow. Every month that passes with teams shipping agent-generated code they do not understand is another month of cognitive debt accumulation. Unlike technical debt, cognitive debt does not send warning signals. It feels like productivity right up until the moment it feels like a crisis.

Daniel Stenberg, the creator of cURL, issued the most prescient warning at FOSDEM 2026: "Everyone is going to vibe code their own. And then, at the end of 2026, everyone is going to start realising what kind of burden they actually built themselves into."[^1]

We do not have to wait until the end of 2026 to see it. The teams that separate specification from implementation, that enforce architectural coherence through AGENTS.md, that require human understanding before merge, that measure cognitive debt explicitly — those teams will capture the full productivity of AI agents without the hidden liability.

The rest will spend Q4 in a forensic archaeology expedition through their own codebase, trying to understand code that nobody wrote and nobody reviewed, wondering how it got this bad this fast.

The agents are not the problem. The absence of engineering discipline around the agents is the problem. And unlike the agents themselves, engineering discipline does not emerge from a prompt.

It emerges from the decision your team makes this week about whether "tests pass" is sufficient, or whether understanding is still required.

Choose understanding. The codebase you save will be your own.

The risk is named. Now we turn to understanding the machinery that can contain it. In [Article 08: Inside the Machine](/premium/08-inside-the-machine/), we open the engine — a deep dive into Codex CLI's internals so you can tune, extend, and trust the factory's core infrastructure.

## Citations

## The Agentic Engineering Series {#series}

From experiment to enterprise — building the factory for AI-assisted software engineering at scale.

| | Article | Role |
|---|---------|------|
| 1 | [Agentic Engineering Is Not Vibe Coding](/premium/01-agentic-engineering-is-not-vibe-coding/) | The Wake-Up Call |
| 2 | [Codex CLI at One Year](/premium/02-codex-cli-at-one-year/) | The Platform |
| 3 | [The AGENTS.md Playbook](/premium/03-the-agents-md-playbook/) | The Blueprint |
| 4 | [TDAD and the Testing Revolution](/premium/04-tdad-and-the-testing-revolution/) | The Quality Gate |
| 5 | [The Agentic Pod](/premium/05-the-agentic-pod/) | The Team Model |
| 6 | [Three Terminals, Three Fates](/premium/06-three-terminals-three-fates/) | The Toolchain |
| **7** | **[AI Slopageddon](/premium/07-ai-slopageddon/)** | **The Risk** |
| 8 | [Inside the Machine](/premium/08-inside-the-machine/) | The Engine |
| 9 | [Complete Guide to Codex Security](/premium/09-complete-guide-to-codex-security/) | The Guardrails |
| 10 | [Context Compaction and Memory](/premium/10-context-compaction-and-memory/) | The Efficiency Layer |
| 11 | [Token Economics and ROI](/premium/11-token-economics-and-the-roi-of-coding-agents/) | The Business Case |
| 12 | [The Scaling Playbook](/premium/12-the-scaling-playbook/) | The Rollout |
| 13 | [The Agentic Engineering Maturity Matrix](/premium/13-the-agentic-engineering-maturity-matrix/) | The Assessment |

[^1]: Stenberg, D. (2026). "AI-generated bug reports and the curl bug bounty shutdown." curl blog and FOSDEM 2026 presentation. Also covered in InfoQ: "AI Vibe Coding Threatens Open Source." https://www.infoq.com/news/2026/02/ai-floods-close-projects/

[^2]: Byteable. (2026). "Top AI Code Refactoring Tools for Tackling Technical Debt in 2026." https://www.byteable.ai/blog/top-ai-code-refactoring-tools-for-tackling-technical-debt-in-2026

[^3]: Beck, K., interviewed by Orosz, G. (2025). "TDD, AI Agents, and Coding with Kent Beck." The Pragmatic Engineer, June 11, 2025. https://newsletter.pragmaticengineer.com/p/tdd-ai-agents-and-coding-with-kent

[^4]: Turing College. (2026). "Agentic Engineering vs. Vibe Coding." https://www.turingcollege.com/blog/agentic-engineering-vs-vibe-coding

[^5]: Bara, M. (2026). "AI Is the Largest Consumer of Open Source in History, and Its Worst Contributor." Medium. https://medium.com/@marc.bara.iniesta/ai-is-the-largest-consumer-of-open-source-in-history-and-its-worst-contributor-91bc776438b5

[^6]: Vaughan, D. (2026). "Vibe Coding vs Agentic Engineering: A Senior Developer's Framework." Codex Resources. https://codex.danielvaughan.com/articles/2026-03-29-vibe-coding-vs-agentic-engineering/

[^7]: OpenAI. (2026). "Best practices for Codex CLI — AGENTS.md as durable project guidance." https://developers.openai.com/codex/learn/best-practices

[^8]: OpenAI. (2026). "Codex CLI Features: /review command and review_model configuration." https://developers.openai.com/codex/cli/features

[^9]: Every, Inc. (2026). "Compound Engineering: How Every Codes With Agents." https://every.to/chain-of-thought/compound-engineering-how-every-codes-with-agents

[^10]: Karpathy, A. (2026). One-year anniversary thread renaming "vibe coding" to "agentic engineering." February 2026. https://x.com/karpathy/status/2019137879310836075

[^11]: Osmani, A. (2026). Quoted in "It's Not Vibe Coding: Agentic Engineering." Michael Kennedy's blog. https://mkennedy.codes/posts/its-not-vibe-coding-agentic-engineering/

[^12]: This is a composite scenario based on patterns observed across multiple engineering teams, not a single incident. The details are representative, not literal.
