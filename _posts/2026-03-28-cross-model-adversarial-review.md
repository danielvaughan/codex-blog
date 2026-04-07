---
title: "Cross-Model Adversarial Review: Using Multiple AI Models to Catch Agent Blind Spots"
date: 2026-03-28
tags: [multi-agent, code-review, adversarial, cross-model, quality-gates]
---

![Sketchnote diagram for: Cross-Model Adversarial Review: Using Multiple AI Models to Catch Agent Blind Spots](/sketchnotes/articles/2026-03-28-cross-model-adversarial-review.png)


*Published: 2026-03-28*

The moment your coding agent reviews its own output, you have a problem. Not because the agent is dishonest — but because it is architecturally incapable of neutrality. The same model that rationalised a design shortcut during implementation will rationalise it again during review. This is not a limitation that better prompting can fix. It requires a structural solution: a different model doing the reviewing.

## Why Same-Model Self-Review Fails

LLMs struggle with self-validation. They tend to "hallucinate correctness" — reproducing the same reasoning errors they made during generation. A model reviewing its own code is not checking the code against the spec; it is checking whether the code looks like it was generated correctly.

The result is a class of bugs that consistently survive same-model review:
- **Silent violations** — code that passes tests but violates architectural constraints (e.g., filtering in memory instead of at the database layer)
- **Spec drift** — requirements that were satisfied in the first iteration but quietly regressed
- **Security blind spots** — patterns the generating model treats as standard practice (global state, shared mutable structures, weak input sanitisation)
- **Edge-case omissions** — paths the model never considered during generation, and therefore never checks for during review

One practitioner described the realisation clearly: *"you need to have your agents adversarially reviewed… not having Claude review Claude's output — have Codex and Gemini review Claude's output. So every time my Claude Code sessions do anything, they commit and the commit is immediately reviewed by either Codex or Gemini or both."*

## The Builder-Critic Architecture

The solution is structural. Two roles, different models, clean context separation.

```
┌─────────────────────────────────────────────────────┐
│                ADVERSARIAL REVIEW LOOP              │
│                                                     │
│  ┌──────────┐    implements    ┌────────────────┐   │
│  │  SPEC /  │ ──────────────► │  BUILDER       │   │
│  │  TESTS   │                 │  (Codex CLI)   │   │
│  └──────────┘                 └───────┬────────┘   │
│                                       │             │
│                               code diff             │
│                                       │             │
│                               ┌───────▼────────┐   │
│                               │  CRITIC        │   │
│                               │  (Claude Code) │   │
│                               └───────┬────────┘   │
│                                       │             │
│                              PASS / violations      │
│                                       │             │
│                         ┌─────────────▼──────────┐ │
│                         │  MODERATOR (optional)  │ │
│                         │  deduplicates findings │ │
│                         └────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

**Builder role** — optimised for implementation speed. Receives the spec, generates code, writes tests. Typically Codex CLI (`--model gpt-5.4` for routine tasks, `codex-spark` for fast iteration).

**Critic role** — optimised for reasoning depth. Receives the spec and the code diff in a **fresh context** (no memory of the build). Produces `PASS` or an ordered list of violations with line references. Typically Claude Code or a high-reasoning Codex session.

**Critical step — context swap**: Start a new session for the Critic. Do not share conversation history from the build phase. This forces the Critic to evaluate only the artifacts, not the reasoning that produced them.

## Model Routing in Practice

| Role | Good model choice | Why |
|------|-------------------|-----|
| Builder (routine) | `codex-spark` / `gpt-5.4-mini` | High throughput, low cost |
| Builder (complex) | `gpt-5.4` with `reasoning: high` | Better architectural decisions |
| Critic | Claude Code (`claude-opus-4.x`) | Strong constraint validation, different training distribution |
| Critic (fast) | Gemini CLI | Different architecture catches different patterns |
| Moderator | Any fast model | Deduplication only, low reasoning load |

The key principle: **use models from different training distributions**. A Claude-reviewed Codex PR is more reliable than a Codex-reviewed Codex PR — not because Claude is "better", but because it was trained on different data with different biases.

## Implementing the Pattern in AGENTS.md

```markdown
## Review Policy

All PRs require cross-model adversarial review before human review:

1. Builder subagent implements (Codex CLI, model: codex-spark)
2. All tests must pass locally before review step
3. Critic runs in a FRESH session (no shared history):
   - Feed: spec + test file + code diff only
   - Model: claude-opus or gemini (NOT the same model that built)
   - Task: list spec violations with file:line evidence. PASS or FAIL verdict.
4. On FAIL: Builder fixes, re-validates, spawns a NEW critic (not same session)
5. After 3 failures: escalate to human

Never use the same model as both builder and critic.
Never reuse a critic session — anchoring bias is real, even for AI.
```

## The Debate Loop Pattern (alecnielsen/adversarial-review)

For high-stakes changes, a multi-round debate produces better results than a single-pass review:

```
Round 1: Claude and Codex independently review the same diff
Round 2: Each agent critiques the other's findings
Round 3: Each agent responds to critiques
Round 4: One agent synthesises and implements validated fixes
```

The circuit breaker matters: stop if no progress after 3 iterations, or if disagreement persists after 5. Infinite debate loops are a real failure mode.

**Approximate cost**: ~21 API calls worst-case for three iterations. High, but justified for changes touching authentication, data migrations, or public APIs.

## Metaswarm's Industrial-Scale Approach

[Metaswarm](https://github.com/dsifry/metaswarm) implements adversarial review at framework scale. Every work unit runs through a four-phase loop:

```
IMPLEMENT → VALIDATE → ADVERSARIAL REVIEW → COMMIT
```

Key rules:
- Writer is always reviewed by a different model (Codex implements, Claude reviews — or vice versa)
- Fresh reviewer on every retry — the reviewer checks the contract, not "did they fix what I found?"
- Orchestrator independently verifies results (never trusts subagent self-reports)
- Parallel Design Review Gate: 5 specialist agents (PM, Architect, Designer, Security, CTO) reviewing in parallel before implementation begins

The parallel design review gate is worth noting: five specialist personas reviewing a spec simultaneously, each bringing a different adversarial lens. The overlap in findings increases confidence; the disagreements surface spec ambiguities before a line of code is written.

## CI/CD Integration

Adversarial review works as a CI gate:

```yaml
# .github/workflows/adversarial-review.yml
name: Adversarial Code Review
on: [pull_request]
jobs:
  critic:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run critic agent
        run: |
          # Pass spec + diff to a different model than the one that generated the PR
          codex exec "You are a Critic agent. Review this diff against SPEC.md.
          List all spec violations with file:line evidence. Output PASS or FAIL." \
            --model claude-proxy \  # route to Claude for cross-model review
            --approval-mode full-auto \
            --output-format json > review.json

          # Fail the PR if FAIL verdict
          jq -e '.verdict == "PASS"' review.json
```

*Note: Claude proxy routing requires an MCP bridge or custom shell wrapper since `codex exec` natively uses OpenAI models.*

## When Same-Model Review is Acceptable

Same-model review is not always wrong. It is acceptable for:
- **Small, factual diffs** — renaming a variable, updating a config value, fixing a typo
- **Style/format checks** — lint-level concerns with no architectural implications
- **Documentation review** — lower stakes, less likely to have hidden logic errors

It is unacceptable for:
- Any change touching authentication, authorisation, or data handling
- Database schema changes or migrations
- Public API changes
- Code that's been failing tests and has been "fixed" by an agent

## Cost Tradeoffs

Cross-model adversarial review is not cheap. The overhead is real:
- Two model calls instead of one for every reviewed change
- Moderate cost for debate loops (3-5 rounds × 2 models)
- Latency penalty: critics add 30-90 seconds per review depending on diff size

The calculation: one production security incident costs more than months of adversarial review overhead. For changes where mistakes are expensive, the cost is justified. For routine maintenance, skip it.

## Tools

| Tool | URL | Approach |
|------|-----|----------|
| alecnielsen/adversarial-review | https://github.com/alecnielsen/adversarial-review | Claude + GPT Codex debate loop, bash script, multi-round |
| dsifry/metaswarm | https://github.com/dsifry/metaswarm | Full framework with adversarial review built into SDLC |
| codexstar69/bug-hunter | https://github.com/codexstar69/bug-hunter | Hunter + Skeptic + Referee tri-agent security review |
| asdlc.io adversarial-code-review | https://asdlc.io/patterns/adversarial-code-review/ | Pattern documentation with Critic prompt examples |

## Key Takeaways

- Same-model self-review perpetuates the same blind spots — structural fix required
- Always start the Critic in a **fresh context** — conversation history from the build phase contaminates the review
- Use models from **different training distributions** for genuine adversarial benefit
- The writer-reviewer rule: the model that wrote the code **must not** review it
- On retry: always spawn a **new reviewer session** — anchoring bias affects AI as well as humans
- For CI/CD: adversarial review as a gate, not an afterthought

*Sources: [asdlc.io adversarial code review](https://asdlc.io/patterns/adversarial-code-review/), [metaswarm](https://github.com/dsifry/metaswarm), [alecnielsen/adversarial-review](https://github.com/alecnielsen/adversarial-review), [halallens.no multi-model guide](https://halallens.no/en/blog/agentic-coding-in-2026-the-complete-guide-to-plugins-multi-model-orchestration-and-ai-agent-teams)*
