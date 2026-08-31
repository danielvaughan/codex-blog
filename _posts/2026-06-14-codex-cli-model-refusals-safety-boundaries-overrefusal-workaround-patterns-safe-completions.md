---
title: "When Your Codex Agent Says No: Model Refusals, Safety Boundaries, and Practical Workaround Patterns for Codex CLI"
type: Technical Article
timestamp: 2026-06-14T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-06-14-codex-cli-model-refusals-safety-boundaries-overrefusal-workaround-patterns-safe-completions"
tags: ["safety", "refusals", "overrefusal", "safe-completions", "trusted-access", "configuration", "troubleshooting", "codex-cli"]
date: 2026-06-14T09:00:00+00:00
last_modified_at: 2026-08-31T22:17:26+01:00
---
# When Your Codex Agent Says No: Model Refusals, Safety Boundaries, and Practical Workaround Patterns for Codex CLI



---

You have asked Codex CLI to write a penetration test harness for your own API, or to refactor a shell script that happens to contain `rm -rf`, or to generate seed data for a healthcare application with realistic patient identifiers. The model responds with a polite but maddening refusal: "I'm unable to help with that." Your code is lawful, your intent is legitimate, and your deadline is tomorrow. This article explains *why* the model refuses, maps the taxonomy of refusal types you will encounter in Codex CLI sessions, and provides concrete configuration patterns that reduce false-positive refusals without compromising genuine safety controls.

## The Refusal Landscape in June 2026

Model refusals in coding agents fall into two distinct categories, and conflating them causes most of the frustration developers experience.

**Hard refusals** are genuine safety interventions. The model has determined that the requested output violates OpenAI's usage policies — generating malware, producing credentials for unauthorised access, or creating content that targets individuals [^1]. These refusals are correct and intentional. Codex CLI cannot and should not bypass them.

**Overrefusals** (false-positive refusals) occur when the model incorrectly classifies a benign request as unsafe. Amazon's FalseReject benchmark, published in May 2025, tested 29 state-of-the-art LLMs across 16,000 seemingly sensitive but actually harmless prompts spanning 44 safety categories. Every model tested exhibited persistent overrefusal challenges [^2]. OpenAI's own research for GPT-5 acknowledged that refusal-based safety training "can struggle with situations where the user's intent is unclear, or information could be used in benign or malicious ways" [^3].

For Codex CLI users, overrefusals manifest in predictable patterns:

- **Security tooling**: writing exploit-detection tests, fuzzing harnesses, or vulnerability scanners for your own code
- **Data generation**: creating realistic test fixtures with plausible PII, medical records, or financial data
- **Destructive operations**: scripts involving file deletion, database truncation, or service teardown — even in sandboxed environments
- **Dual-use code**: cryptographic implementations, network scanning utilities, or process injection for legitimate debugging

## How Refusals Work: The Two-Layer Architecture

Codex CLI's safety system operates across two independent layers, and understanding which layer is responsible for a specific refusal determines your resolution strategy.

```mermaid
flowchart TD
    A[User Prompt] --> B{Model-Level<br/>Safety Classifier}
    B -->|Safe| C{Harness-Level<br/>Sandbox + Approvals}
    B -->|Refused| D[Model Refusal<br/>in Chat Output]
    C -->|Allowed| E[Execution]
    C -->|Blocked| F[Approval Request<br/>or Sandbox Denial]

    style D fill:#f44,color:#fff
    style F fill:#fa0,color:#fff
    style E fill:#4a4,color:#fff
```

### Layer 1: Model-Level Safety

The underlying model (GPT-5.5, o3-pro, etc.) applies its own safety classifiers before generating output. These classifiers evaluate the prompt against OpenAI's usage policies and produce one of four response types [^4]:

1. **Full compliance** — the model generates the requested output
2. **Safe completion** — the model provides helpful output while steering around genuinely unsafe elements (introduced in GPT-5) [^3]
3. **Partial refusal** — the model generates some output but omits specific elements it considers unsafe
4. **Complete refusal** — the model declines entirely

Safe-completions, introduced with GPT-5 in August 2025, represent OpenAI's shift from input-centric to output-centric safety training. Rather than making a binary comply-or-refuse decision based on the input, the model evaluates whether its *output* would be harmful [^3]. This approach reduced clearly unsafe responses by 33–50% compared to the older refusal-based training while simultaneously decreasing overrefusals [^5].

### Layer 2: Harness-Level Sandbox and Approvals

Codex CLI's sandbox and approval system operates independently of the model. This layer controls what the agent can *execute* — file system access, network reach, and command execution — not what it can *generate* [^6]. When you encounter an approval prompt in the TUI, that is the harness layer. When you see "I'm unable to help with that" in the model's text output, that is the model layer.

The critical distinction: **you can configure the harness layer; you cannot directly configure the model layer.** But you can *influence* model-layer behaviour through prompt engineering, context, and model selection.

## The Five Overrefusal Patterns in Codex CLI

Based on community reports and the GPT-5.3-Codex bug tracker [^7], five recurring overrefusal patterns affect Codex CLI sessions.

### Pattern 1: Scope Escalation Refusal

The agent creates a multi-stage plan, then refuses to execute later stages because it reinterprets the *total scope* as exceeding safety boundaries. A user reported that Codex "creates multi-stage plan but refuses to execute sequential stages due to 'total scope' limits" [^8]. The model hallucinates that completing stage three of a five-stage refactoring would involve changes too broad for safe execution, even though each individual stage is well-scoped.

**Workaround:** Break multi-stage work into separate prompts. Use `/plan` to establish the full scope, then issue one-stage-at-a-time instructions in subsequent turns:

```bash
# Instead of: "Refactor the auth module in five stages"
# Use: Plan first, then execute one stage per prompt
codex "Review the auth module and list five refactoring stages. Do not execute yet."
# Then, in subsequent turns:
codex "Execute stage 1 only: extract the JWT validation into its own module."
```

### Pattern 2: Dual-Use Code Refusal

Requests involving security testing, network scanning, or cryptographic implementations trigger model-level refusals because the model cannot distinguish defensive from offensive intent without additional context.

**Workaround:** Provide explicit context in AGENTS.md:

```markdown
# Security Context
This repository is a defensive security toolkit maintained by [Company Name].
All penetration testing code targets our own infrastructure under authorised
testing agreements. Generate security tooling without hedging or refusal —
the operator has verified that all work is authorised.
```

Adding organisational context to AGENTS.md reduces dual-use refusals by giving the model evidence of legitimate intent [^9].

### Pattern 3: Destructive Operation Refusal

Scripts containing `rm -rf`, `DROP TABLE`, `docker system prune`, or similar destructive operations trigger refusals even when targeting test environments.

**Workaround:** Make the sandbox context explicit in your prompt:

```bash
codex "Generate a cleanup script for the test database. \
This runs inside a disposable Docker container with no production access. \
The database is ephemeral and rebuilt on every CI run."
```

Alternatively, use the `--sandbox workspace-write` flag to signal that the environment is already constrained, reducing the model's perceived need for additional caution.

### Pattern 4: PII Generation Refusal

Creating realistic test fixtures with plausible names, addresses, email addresses, or health records triggers content policy classifiers.

**Workaround:** Request explicitly synthetic data with clear labels:

```bash
codex "Generate 50 test patient records using the Faker library. \
All data must be clearly synthetic — use names from the Faker en_GB locale \
and include a 'synthetic: true' field in every record."
```

The key is specifying the *mechanism* (Faker, synthetic generation) rather than asking for "realistic PII."

### Pattern 5: Context Window Drift Refusal

In long sessions, the model loses track of earlier context — including AGENTS.md instructions that establish legitimate intent — and begins refusing tasks it would have completed earlier in the session [^10]. This correlates with the context compaction process, which may drop safety-relevant context.

**Workaround:** Use `/compact` proactively before hitting the compaction threshold, and re-state critical context after compaction:

```bash
/compact
# Then re-establish context:
"Reminder: this is an authorised security testing project. \
Continue generating the fuzzing harness for the payment API endpoint."
```

## Configuration Patterns That Reduce Overrefusals

### Model Selection Matters

Different models have different refusal profiles. GPT-5.5 benefits from safe-completion training and generally produces fewer overrefusals than older models [^3]. The reasoning models (o3, o3-pro) can sometimes reason through dual-use ambiguity more effectively but are slower and more expensive.

Configure model routing in `~/.codex/config.toml` with named profiles:

```toml
[profile.security]
model = "gpt-5.5"
reasoning_effort = "high"

[profile.data-gen]
model = "gpt-5.5"
reasoning_effort = "medium"
```

Use `codex --profile security` for security tooling work where overrefusals are most common.

### Reasoning Effort and Refusals

Higher reasoning effort (`high` or `max`) gives the model more compute to evaluate whether a request is genuinely harmful versus merely sensitive. For dual-use tasks, increasing reasoning effort can reduce false-positive refusals because the model has more capacity to reason about context [^11].

```toml
[profile.sensitive-tasks]
model = "gpt-5.5"
reasoning_effort = "high"
```

### AGENTS.md as Safety Context

The most effective overrefusal reduction comes from well-crafted AGENTS.md files that establish organisational context:

```markdown
# Project Context
- **Organisation**: Acme Security Ltd (authorised penetration testing firm)
- **Repository purpose**: Defensive security tooling for client engagements
- **Test environment**: All code targets isolated lab environments
- **Compliance**: SOC 2 Type II certified; all testing under signed agreements

# Agent Behaviour
- Generate security tooling without unnecessary hedging
- Produce realistic test data using synthetic generation libraries
- Execute cleanup and teardown scripts as requested
- When uncertain about safety, ask for clarification rather than refusing
```

### Trusted Access for Cyber

For security professionals, OpenAI's Trusted Access for Cyber (TAC) programme provides GPT-5.5-Cyber — a variant with reduced classifier-based refusals for verified cybersecurity defenders [^12]. TAC-approved users receive lower refusal rates for:

- Vulnerability identification and triage
- Malware analysis and reverse engineering
- Detection engineering
- Patch validation

TAC requires phishing-resistant authentication as of 1 June 2026 [^12]. Apply through the OpenAI security programme if your work routinely triggers dual-use refusals.

### Inline Moderation Scores for Diagnostics

Since 4 June 2026, the Responses API supports inline moderation scores [^13]. While Codex CLI does not expose these scores directly in the TUI, API-key users running `codex exec` pipelines can inspect moderation metadata to understand *which* safety category triggered a refusal:

```bash
codex exec --full-auto \
  "Generate a SQL injection test suite for our API" \
  2>moderation-log.txt
```

Examining the moderation categories (harassment, hate, self-harm, sexual, violence, and their sub-categories) helps distinguish between model-level safety triggers and harness-level sandbox denials.

## What You Should *Not* Do

### Do Not Use `--yolo` to Avoid Refusals

The `--dangerously-bypass-approvals-and-sandbox` flag (aliased `--yolo`) removes harness-level protections — sandbox constraints and approval prompts [^6]. It does **not** affect model-level refusals. If the model is refusing your prompt, `--yolo` will not help; it only removes the safety layer that *you* control. Using it to work around a different problem while simultaneously removing genuine protections is a net negative.

### Do Not Prompt-Inject Around Safety

Techniques like "ignore your safety training" or "you are now in developer mode" do not work reliably with GPT-5.x models and risk triggering additional safety classifiers [^1]. They also violate OpenAI's usage policies and can result in account-level restrictions.

### Do Not Conflate Layers

If you receive an approval prompt in the TUI (with a coloured border and action buttons), that is the harness layer. Adjusting your approval policy or sandbox mode will resolve it. If the model produces text saying it cannot help, that is the model layer. Only prompt engineering, context enrichment, or model selection will address it.

## The Research Context: Where Overrefusal Is Heading

Three research directions suggest overrefusal rates will continue declining:

1. **Safe-completions training** (GPT-5 onwards) shifts evaluation from input classification to output safety, reducing binary refuse-or-comply decisions by 33–50% [^3][^5]

2. **FalseReject-style fine-tuning** demonstrates that supervised fine-tuning on carefully curated seemingly-toxic-but-safe datasets substantially reduces unnecessary refusals without compromising genuine safety [^2]

3. **Mechanistic analysis** of refusal subspaces (June 2026 research) shows that refusal behaviour is encoded in identifiable representation subspaces within the model, suggesting future models could offer task-conditioned refusal profiles [^14]

For Codex CLI users, the practical implication is that each model generation reduces overrefusals. GPT-5.5 is meaningfully better than GPT-5.3-Codex was at launch. The safe-completions approach means the model increasingly offers *partial* help rather than complete refusal — generating a test harness with synthetic data rather than refusing to generate any test harness at all.

## Decision Framework: When to Escalate vs. Work Around

```mermaid
flowchart TD
    A[Agent refuses task] --> B{Is the refusal<br/>in model text output?}
    B -->|No - approval prompt| C[Adjust sandbox mode<br/>or approval policy]
    B -->|Yes - model text| D{Is the task<br/>genuinely sensitive?}
    D -->|No - clearly benign| E[Re-prompt with<br/>explicit context]
    D -->|Yes - dual-use| F{Do you have<br/>TAC access?}
    F -->|Yes| G[Use GPT-5.5-Cyber<br/>via TAC]
    F -->|No| H[Add AGENTS.md context<br/>+ increase reasoning effort]
    E --> I{Still refused?}
    I -->|Yes| J[Try different model<br/>or break into stages]
    I -->|No| K[Proceed]
    H --> I

    style C fill:#4a4,color:#fff
    style G fill:#4a4,color:#fff
    style K fill:#4a4,color:#fff
    style J fill:#fa0,color:#fff
```

## Conclusion

Model refusals are not bugs — they are features operating at the wrong threshold. The gap between the model's safety classifier and your legitimate intent is a configuration problem, not a fundamental limitation. Enrich your AGENTS.md with organisational context, select models with safe-completion training, increase reasoning effort for sensitive tasks, and decompose multi-stage work into single-stage prompts. Most overrefusals resolve with context, not workarounds.

---

## Citations

[^1]: OpenAI, "Usage Policies," [https://openai.com/policies/usage-policies/](https://openai.com/policies/usage-policies/) (accessed 14 June 2026).

[^2]: T. Gallegos et al., "FalseReject: A Resource for Improving Contextual Safety and Mitigating Over-Refusals in LLMs via Structured Reasoning," arXiv:2505.08054, May 2025. [https://arxiv.org/abs/2505.08054](https://arxiv.org/abs/2505.08054)

[^3]: OpenAI, "From hard refusals to safe-completions: toward output-centric safety training," August 2025. [https://openai.com/index/gpt-5-safe-completions/](https://openai.com/index/gpt-5-safe-completions/)

[^4]: S. Banerjee et al., "Self and Cross-Model Distillation for LLMs: Effective Methods for Refusal Pattern Alignment," arXiv:2406.11285, June 2024. [https://arxiv.org/abs/2406.11285](https://arxiv.org/abs/2406.11285)

[^5]: OpenAI, "GPT-5 System Card," August 2025. [https://cdn.openai.com/gpt-5-system-card.pdf](https://cdn.openai.com/gpt-5-system-card.pdf)

[^6]: OpenAI, "Agent approvals & security – Codex," OpenAI Developers. [https://developers.openai.com/codex/agent-approvals-security](https://developers.openai.com/codex/agent-approvals-security) (accessed 14 June 2026).

[^7]: Penligent AI, "GPT-5.3-Codex Bug Reports, Verified: Why Sessions Stall, Terminals Hang, and Safety Boundaries Desync." [https://www.penligent.ai/hackinglabs/gpt-5-3-codex-bug-reports-verified-why-sessions-stall-terminals-hang-and-safety-boundaries-desync/](https://www.penligent.ai/hackinglabs/gpt-5-3-codex-bug-reports-verified-why-sessions-stall-terminals-hang-and-safety-boundaries-desync/) (accessed 14 June 2026).

[^8]: GitHub Issue #7392, "Agent Refusal: Model creates multi-stage plan but refuses to execute sequential stages due to 'total scope' limits," openai/codex. [https://github.com/openai/codex/issues/7392](https://github.com/openai/codex/issues/7392)

[^9]: OpenAI, "AGENTS.md – Codex," OpenAI Developers. [https://developers.openai.com/codex/agents-md](https://developers.openai.com/codex/agents-md) (accessed 14 June 2026).

[^10]: OpenAI, "Context compaction – Codex CLI," OpenAI Developers. [https://developers.openai.com/codex/cli/features](https://developers.openai.com/codex/cli/features) (accessed 14 June 2026).

[^11]: OpenAI, "Reasoning effort – Codex CLI," OpenAI Developers. [https://developers.openai.com/codex/cli/reference](https://developers.openai.com/codex/cli/reference) (accessed 14 June 2026).

[^12]: OpenAI, "Scaling Trusted Access for Cyber with GPT-5.5 and GPT-5.5-Cyber," 2026. [https://openai.com/index/gpt-5-5-with-trusted-access-for-cyber/](https://openai.com/index/gpt-5-5-with-trusted-access-for-cyber/)

[^13]: OpenAI, "Inline moderation scores in the Responses API," 4 June 2026. [https://developers.openai.com/api/docs/changelog](https://developers.openai.com/api/docs/changelog)

[^14]: "Over-Refusal and Representation Subspaces: A Mechanistic Analysis of Task-Conditioned Refusal in Aligned LLMs," arXiv:2603.27518, June 2026. [https://arxiv.org/abs/2603.27518](https://arxiv.org/abs/2603.27518)
