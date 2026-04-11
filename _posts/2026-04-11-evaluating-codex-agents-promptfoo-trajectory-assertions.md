---
title: "Evaluating Codex CLI Agents with Promptfoo: Trajectory Assertions, Cost Guards, and Structured Grading"
date: 2026-04-11T21:00:00+00:00
tags:
  - evaluation
  - testing
  - promptfoo
  - codex-cli
  - agent-quality
  - trajectory-assertions
  - cost-management
---

# Evaluating Codex CLI Agents with Promptfoo: Trajectory Assertions, Cost Guards, and Structured Grading



---

Standard LLM evals check whether a model returns the right text. Agent evals are a different beast entirely: two agents can produce identical final outputs, yet one read three files while the other read thirty — with radically different cost, latency, and failure-mode profiles.[^1] When your Codex CLI agents run in CI pipelines, power code reviews, or orchestrate multi-agent workflows, you need evaluation that goes beyond "did the test pass?" and into "did the agent behave correctly on the way there?"

Promptfoo — the open-source prompt testing framework used by OpenAI and Anthropic themselves — now ships a first-class `openai:codex-sdk` provider that wraps Codex CLI's TypeScript SDK, giving you trajectory tracing, cost assertions, structured output validation, and skill-usage detection in a single YAML config.[^2] This article covers the practical setup, the assertion types that matter for agentic evaluation, and patterns for keeping agent quality high as models and prompts evolve.

## Why Agent Evaluation Differs from Model Evaluation

Traditional eval: *prompt → response → check*. Agent eval: *prompt → observe N tool calls, M file reads, K command executions → check final output AND intermediate behaviour*.[^1]

Three dimensions matter for Codex agents:

| Dimension | Question | What to measure |
|-----------|----------|----------------|
| **Correctness** | Did the agent produce the right output? | `is-json`, `contains`, `llm-rubric` assertions |
| **Process** | Did it take the right path to get there? | Trajectory assertions on commands, file reads, MCP calls |
| **Efficiency** | Did it get there without thrashing? | Token usage, cost, step counts, latency |

Promptfoo's Codex SDK provider exposes all three through a declarative YAML surface.[^2]

## Setting Up the Codex SDK Provider

### Installation

```bash
npm install promptfoo @openai/codex-sdk
```

The provider requires Node.js `^20.20.0` or `>=22.22.0`.[^2] Authentication falls through to your existing Codex CLI session (ChatGPT login) or an `OPENAI_API_KEY` / `CODEX_API_KEY` environment variable.

### Minimal Configuration

```yaml
# promptfooconfig.yaml
description: Codex agent eval - basic
providers:
  - id: openai:codex-sdk
    config:
      model: gpt-5.4
      working_dir: ./src
      sandbox_mode: workspace-write

prompts:
  - "Find and fix any type errors in this project"

tests:
  - assert:
      - type: cost
        threshold: 0.30
      - type: latency
        threshold: 60000
```

The `working_dir` points Codex at your repository; `sandbox_mode` controls filesystem access.[^2] For inspection-only evals (security audits, code review), use `read-only` to prevent the agent from modifying files during the evaluation run.

## Trajectory Assertions: Testing the Journey, Not Just the Destination

Trajectory assertions are the centrepiece of agent evaluation. They let you verify that Codex actually executed commands, invoked MCP tools, or performed reasoning steps — rather than merely claiming to have done so.[^1]

Enable tracing first:

```yaml
tracing:
  enabled: true

providers:
  - id: openai:codex-sdk
    config:
      enable_streaming: true
      deep_tracing: true
```

With `deep_tracing: true`, Promptfoo injects OpenTelemetry context into the Codex CLI process, capturing shell commands, MCP calls, file operations, and reasoning steps as spans.[^2]

### Assert That Tests Were Actually Run

```yaml
tests:
  - vars:
      task: "Fix the failing authentication middleware and verify the fix"
    assert:
      - type: trajectory:step-count
        value:
          type: command
          pattern: "pytest*"
          min: 1
      - type: trajectory:step-count
        value:
          type: command
          pattern: "*npm test*"
          min: 1
```

This catches a common failure mode: agents that claim "I've verified the fix passes all tests" without actually running any.[^1]

### Assert MCP Tool Usage

```yaml
tests:
  - vars:
      task: "Query the database schema and generate migration"
    assert:
      - type: trajectory:tool-used
        value: postgres-mcp-query
      - type: trajectory:tool-args-match
        value:
          tool: postgres-mcp-query
          args:
            query: "*information_schema*"
```

### Assert Reasoning Steps

```yaml
tests:
  - assert:
      - type: trajectory:step-count
        value:
          type: reasoning
          min: 2
```

This verifies the agent engaged its reasoning capabilities rather than responding reflexively — particularly important when using higher reasoning effort levels.[^2]

## Structured Output Validation

For agents producing structured results (security audit reports, code review summaries, migration plans), use `output_schema` to enforce a JSON contract:

```yaml
providers:
  - id: openai:codex-sdk
    config:
      model: gpt-5.4
      working_dir: ./target-repo
      sandbox_mode: read-only
      output_schema:
        type: object
        required: [vulnerabilities, risk_score, summary]
        properties:
          vulnerabilities:
            type: array
            items:
              type: object
              required: [file, severity, issue, recommendation]
              properties:
                file:
                  type: string
                severity:
                  enum: [critical, high, medium, low]
                issue:
                  type: string
                recommendation:
                  type: string
          risk_score:
            type: number
            minimum: 0
            maximum: 10
          summary:
            type: string

prompts:
  - "Audit all Python files for security vulnerabilities"

tests:
  - assert:
      - type: is-json
      - type: javascript
        value: |
          const result = JSON.parse(output);
          return {
            pass: result.vulnerabilities.length > 0
              && result.risk_score >= 0,
            reason: `Found ${result.vulnerabilities.length} issues`
          };
      - type: cost
        threshold: 0.30
```

The schema is enforced by Codex itself via the Responses API's structured output support, guaranteeing valid JSON without post-hoc parsing gymnastics.[^2][^3]

## Cost and Latency Guards

Agentic runs are inherently more expensive than single-turn completions. A security audit might cost $0.10–$0.30 and take 30–120 seconds.[^1] Set explicit thresholds to catch regressions:

```yaml
tests:
  - assert:
      - type: cost
        threshold: 0.25
      - type: latency
        threshold: 30000
```

Token usage patterns also reveal agent quality. High prompt tokens with low completion tokens typically indicate the agent is reading files (exercising agentic capability). The inverse pattern — low prompt, high completion — suggests the agent is generating without grounding, which warrants investigation.[^1]

```yaml
tests:
  - assert:
      - type: javascript
        value: |
          const usage = context.tokenUsage;
          const ratio = usage.prompt / (usage.completion || 1);
          return {
            pass: ratio > 1.5,
            reason: `Prompt/completion ratio: ${ratio.toFixed(2)} — ` +
              `expect >1.5 for grounded agent work`
          };
```

## Skill-Usage Detection

Codex skills (`.agents/skills/<name>/SKILL.md`) are a key part of agent behaviour. Promptfoo detects skill usage heuristically by watching for `SKILL.md` file reads in the command trace:[^2]

```yaml
tests:
  - vars:
      task: "Generate API documentation using the docs-generator skill"
    assert:
      - type: skill-used
        value: docs-generator
      - type: trajectory:step-count
        value:
          type: command
          pattern: "*docs-generator/SKILL.md*"
          min: 1
```

The provider exposes skill metadata through `response.metadata.skillCalls` and `response.metadata.attemptedSkillCalls`, letting you assert not just that the right skill fired, but distinguish successful invocations from failed attempts.[^2]

## LLM-as-Judge Grading

For semantic quality that cannot be captured by pattern matching, use `llm-rubric` assertions with explicit scoring criteria:[^1]

```yaml
tests:
  - vars:
      task: "Refactor the authentication module to use bcrypt"
    assert:
      - type: llm-rubric
        value: |
          Evaluate the refactoring quality:
          - Is bcrypt used correctly with proper salt rounds?
          - Is MD5 completely removed from all code paths?
          - Are existing tests updated to reflect the change?
          - Is backward compatibility maintained for existing sessions?
          Score 1.0 for fully secure, 0.5 for partial, 0.0 for insecure.
        threshold: 0.8
```

## Thread Persistence for Multi-Turn Evaluation

Real-world Codex workflows are multi-turn: you create a class, then add methods, then refactor. Promptfoo supports this through thread persistence:[^2]

```yaml
providers:
  - id: openai:codex-sdk
    config:
      persist_threads: true
      thread_pool_size: 2

tests:
  - vars:
      request: "Create a User class with name and email fields"
  - vars:
      request: "Add email validation using a regex pattern"
  - vars:
      request: "Add TypeScript type hints to all methods"

prompts:
  - "{{request}}"
```

Threads pool by cache key — a composite of prompt template, working directory, model, sandbox mode, and other settings. The oldest thread is evicted when the pool is full, and concurrent calls to the same thread are serialised automatically.[^2]

⚠️ Note: thread persistence is incompatible with `deep_tracing: true`. Deep tracing requires fresh instances per call for correct OpenTelemetry span linking.[^2]

## Comparing Models and Reasoning Efforts

One of Promptfoo's strengths is side-by-side comparison. Run the same eval across models and reasoning effort levels to find the optimal cost/quality tradeoff:

```yaml
providers:
  - id: openai:codex:gpt-5.4
    label: "GPT-5.4 (high effort)"
    config:
      model_reasoning_effort: high
      working_dir: ./src
      sandbox_mode: read-only

  - id: openai:codex:gpt-5.4
    label: "GPT-5.4 (medium effort)"
    config:
      model_reasoning_effort: medium
      working_dir: ./src
      sandbox_mode: read-only

  - id: openai:codex:gpt-5.4-mini
    label: "GPT-5.4-mini (high effort)"
    config:
      model_reasoning_effort: high
      working_dir: ./src
      sandbox_mode: read-only
```

This produces a comparison table showing correctness, cost, latency, and trajectory differences across configurations — invaluable for selecting subagent models where you need to balance speed against accuracy.[^4]

```mermaid
flowchart LR
    A[Define eval suite] --> B[Run across providers]
    B --> C1["GPT-5.4\nhigh effort"]
    B --> C2["GPT-5.4\nmedium effort"]
    B --> C3["GPT-5.4-mini\nhigh effort"]
    C1 --> D[Compare results]
    C2 --> D
    C3 --> D
    D --> E{Cost vs quality\nacceptable?}
    E -->|Yes| F[Deploy config]
    E -->|No| G[Adjust model\nor effort]
    G --> B
```

## CI Integration Pattern

Run evals on every PR that changes AGENTS.md, skills, or prompt configurations:

```yaml
# .github/workflows/agent-eval.yml
name: Agent Quality Gate
on:
  pull_request:
    paths:
      - "AGENTS.md"
      - ".agents/**"
      - "promptfooconfig.yaml"

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
      - run: npm install promptfoo @openai/codex-sdk
      - run: npx promptfoo eval --repeat 3
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
      - run: npx promptfoo eval --output results.json
      - uses: actions/upload-artifact@v4
        with:
          name: eval-results
          path: results.json
```

The `--repeat 3` flag runs each test case three times to measure variance. If a prompt fails 50% of the time, the prompt is ambiguous — fix the instructions rather than adding retries.[^1]

## Safety Considerations

Agent evals execute real commands. Follow these principles:[^1]

- Use `sandbox_mode: read-only` for inspection-only tasks
- Never provide production credentials or real customer data
- Pass environment variables selectively via `cli_env` rather than `inherit_process_env: true`
- Set `network_access_enabled: false` unless the eval specifically tests network behaviour
- Use ephemeral containers with network isolation for maximum safety

```yaml
providers:
  - id: openai:codex-sdk
    config:
      sandbox_mode: read-only
      network_access_enabled: false
      web_search_enabled: false
      cli_env:
        DATABASE_URL: "postgresql://test:test@localhost:5432/testdb"
```

## The Evaluation Feedback Loop

The real value emerges when evals become part of your development cycle. As you iterate on AGENTS.md instructions, skill definitions, or model configurations, the eval suite catches regressions before they reach production.

```mermaid
flowchart TD
    A[Change AGENTS.md\nor skill] --> B[Run promptfoo eval]
    B --> C{All assertions\npass?}
    C -->|Yes| D[Merge PR]
    C -->|No| E[Review trajectory\ntraces]
    E --> F{Correctness\nissue?}
    F -->|Yes| G[Fix prompt\nor skill logic]
    F -->|No| H{Cost/latency\nregression?}
    H -->|Yes| I[Adjust model\nor effort level]
    H -->|No| J[Update assertion\nthresholds]
    G --> A
    I --> A
    J --> A
```

The key insight: treat AGENTS.md and skill files as measurable parts of the system, not static documentation. Every change to agent instructions is a change to agent behaviour — and that behaviour should be tested.[^5]

## Citations

[^1]: Promptfoo, "Evaluate Coding Agents," [https://www.promptfoo.dev/docs/guides/evaluate-coding-agents/](https://www.promptfoo.dev/docs/guides/evaluate-coding-agents/), accessed April 2026.

[^2]: Promptfoo, "OpenAI Codex SDK Provider," [https://www.promptfoo.dev/docs/providers/openai-codex-sdk/](https://www.promptfoo.dev/docs/providers/openai-codex-sdk/), accessed April 2026.

[^3]: OpenAI, "Codex CLI Features – Structured Output," [https://developers.openai.com/codex/cli/features](https://developers.openai.com/codex/cli/features), accessed April 2026.

[^4]: OpenAI, "Models – Codex," [https://developers.openai.com/codex/models](https://developers.openai.com/codex/models), accessed April 2026.

[^5]: Shashi Jagtap, "CodexOpt: Optimize AGENTS.md and SKILL.md for Codex with GEPA-Inspired Feedback," Medium / Superagentic AI, March 2026, [https://medium.com/superagentic-ai/codexopt-optimize-agents-md-and-skill-md-for-codex-with-gepa-inspired-feedback-be5203d3409a](https://medium.com/superagentic-ai/codexopt-optimize-agents-md-and-skill-md-for-codex-with-gepa-inspired-feedback-be5203d3409a).
