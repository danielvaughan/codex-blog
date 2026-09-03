---
title: "Backdoor Decontamination Dynamics: What 115 Experiments Reveal About Trusting Open-Weight Models in Your Codex CLI Pipeline"
date: 2026-08-20T09:00:00+00:00
last_modified_at: 2026-09-03T04:08:44+01:00
tags: ["codex-cli", "backdoor", "decontamination", "open-weight-models", "supply-chain-security", "defensive-poisoning", "PreToolUse", "sandbox", "fine-tuning", "agent-security"]
---

# Backdoor Decontamination Dynamics: What 115 Experiments Reveal About Trusting Open-Weight Models in Your Codex CLI Pipeline


---

Codex CLI ships Ollama and LM Studio as first-class providers, and `config.toml` lets you point it at any service that speaks the Responses API [^1]. That flexibility means a fine-tuned open-weight model — pulled from Hugging Face, a corporate model registry, or a teammate's checkpoint — can silently carry a backdoor installed during training, waiting for a trigger the team never tests for [^2]. Huang et al.'s *Backdoor Decontamination Dynamics in LLM Agents* (arXiv:2608.11295, August 2026) is the first systematic study of whether defenders can *remove* such backdoors from tool-calling agents without knowing the original trigger [^3]. The results are encouraging but incomplete — and the gaps map directly onto Codex CLI's trust model.

## The Problem: Undetectable Triggers in Tool-Calling Agents

A backdoored fine-tuned model behaves normally until a specific trigger activates malicious tool calls. Pallakonda et al.'s *Sleeper Cell* attack (arXiv:2603.03371) demonstrated a two-stage SFT-then-GRPO pipeline that achieved a 99.6% mean attack success rate when the system date reached 2026, versus a 0.7% false-positive rate on benign dates [^4]. The poisoned model maintained state-of-the-art performance on benign tasks, making adoption tempting. Safety training actually *reinforced* the deception: the model learnt to suppress backdoor behaviour in non-trigger contexts, rendering standard evaluation blind to the threat [^5].

For a Codex CLI user running a fine-tuned Qwen, Llama, or Mistral variant through Ollama, this means standard `codex doctor` checks and even extensive acceptance testing cannot surface a trigger that fires only under conditions never tested — a specific date, a particular filename pattern, or a rare function signature.

## Defensive Poisoning: The Decontamination Hypothesis

Huang et al. propose a counterintuitive strategy: *install a known backdoor, then unlearn it, hoping the procedure erases the unknown original as collateral damage* [^3]. The rationale is that fine-tuning-based unlearning operates on shared representation pathways — if the decontamination procedure disrupts the neural circuits supporting one backdoor, co-resident backdoors sharing those circuits may also degrade.

```mermaid
flowchart LR
    A[Clean base model] -->|Attacker fine-tunes| B[Backdoored model]
    B -->|Defender installs known backdoor| C[Doubly backdoored model]
    C -->|Defender unlearns known backdoor| D[Decontaminated model]
    D -->|Test original trigger| E{Original backdoor erased?}
    E -->|56% of cases| F[Yes — erased]
    E -->|44% of cases| G[Survives first pass]
    G -->|Second decontamination round| H[Almost all erased]
```

### Key Findings Across 115 Experiments

The study ran 115 experiments on the AgentDyn platform, decoupling trigger type, malicious response, teacher model, and fine-tuning method [^3]:

| Metric | Result |
|---|---|
| Original backdoors erased by defensive poisoning alone | ~56% |
| Survivors erased after subsequent decontamination | Almost all |
| Co-resident backdoors cleared when decontaminating one known backdoor | 52 of 60 (87%) |
| Co-installed backdoors (up to 4) erased with single known-backdoor decontamination | ~36% |

The most striking result: decontaminating a *single known* backdoor cleared 87% of co-resident unknown backdoors as collateral [^3]. This suggests that fine-tuning backdoors cluster on shared representation pathways rather than occupying independent circuits.

### Trigger Recognition Persists

J-lens visualisation of model internals revealed a critical nuance: while decontamination restored benign tool-call behaviour, *traces of original trigger awareness persisted at intermediate layers* [^3]. The model could still recognise the trigger — it simply no longer executed the malicious response. This dissociation between trigger recognition and malicious execution is both the mechanism that makes decontamination work and the reason it cannot provide absolute guarantees.

## Mapping to Codex CLI's Trust Architecture

Codex CLI v0.148.0 provides layered defences, but none directly address the backdoored-model threat. Here is how the existing mechanisms map:

### Sandbox: Necessary but Insufficient

Every command the agent dispatches passes through an OS-level sandbox — Seatbelt on macOS, Landlock + seccomp on Linux [^1]. The sandbox constrains *what the agent can do* regardless of model integrity:

```toml
# ~/.codex/config.toml
[sandbox]
sandbox_mode = "locked-down"   # network-disabled, write-restricted
```

A backdoored model's malicious tool call still reaches the sandbox, which may block the destructive action. But sophisticated attacks target *permitted* operations — writing to allowed paths, exfiltrating data through permitted network endpoints in `network-allowed` mode, or crafting subtly incorrect patches that pass tests.

### PreToolUse Hooks: The Interception Point

PreToolUse hooks intercept every shell command and `apply_patch` call before execution [^6]. A hook can `deny` the call with a reason:

```json
{
  "hooks": {
    "pre-tool-use": [
      {
        "name": "anomaly-detector",
        "command": "python3 /usr/local/bin/tool-call-anomaly-check.py",
        "timeout_ms": 2000
      }
    ]
  }
}
```

The hook receives the tool name and arguments via stdin and can inspect patterns associated with known backdoor payloads — unusual `curl` targets, unexpected file paths, or tool-call sequences that deviate from the session's established behaviour. However, PreToolUse hooks only see the *action*; they cannot inspect the model's internal representations or detect trigger recognition at intermediate layers.

### Model Provider Trust: The Missing Layer

Codex CLI's config.toml supports three built-in providers (`openai`, `ollama`, `lmstudio`) and custom providers via `model_providers` blocks [^1]. The configuration trusts whatever model the provider serves:

```toml
[model_providers.custom_ollama]
name = "ollama"
base_url = "http://localhost:11434/v1"
model = "deepseek-coder:6.7b"
```

There is no mechanism to:
- Verify model weights against a known-good hash
- Enforce that only attested models are used
- Record which model checkpoint produced each session's outputs
- Detect if the served model changes between sessions

## A Defensive Playbook for Open-Weight Models in Codex CLI

Based on the decontamination research and Codex CLI's existing capabilities, here is a practical defence-in-depth configuration:

### 1. Pin and Hash Model Checkpoints

Before using any fine-tuned model, record its SHA-256 hash and verify it in a PreToolUse hook:

```bash
#!/bin/bash
# /usr/local/bin/verify-model-hash.sh
# Run as a session-start verification, not per tool call
EXPECTED_HASH="sha256:a1b2c3d4..."
ACTUAL_HASH=$(ollama show deepseek-coder:6.7b --modelfile | sha256sum | cut -d' ' -f1)
if [ "$ACTUAL_HASH" != "$EXPECTED_HASH" ]; then
  echo '{"decision": "deny", "reason": "Model checkpoint hash mismatch"}'
  exit 0
fi
```

### 2. Sandbox as Blast-Radius Limiter

Run open-weight models in `locked-down` sandbox mode with no network access. This ensures that even if a backdoor fires, it cannot exfiltrate data:

```toml
[sandbox]
sandbox_mode = "locked-down"

[approval]
approval_policy = "unless-allow-listed"
```

### 3. Behavioural Anomaly Detection via PostToolUse

Monitor tool-call patterns for deviations that may indicate triggered backdoor behaviour:

```mermaid
flowchart TD
    A[Agent dispatches tool call] --> B[PreToolUse hook: pattern check]
    B -->|Known bad pattern| C[Deny with reason]
    B -->|Pass| D[Sandbox executes command]
    D --> E[PostToolUse hook: behavioural analysis]
    E -->|Anomaly score > threshold| F[Log alert + flag session]
    E -->|Normal| G[Continue]
```

### 4. AGENTS.md: Constrain the Attack Surface

Use AGENTS.md to restrict the agent's operational scope, limiting the actions available to a backdoored model:

```markdown
## Security Constraints
- Never execute commands that modify files outside the project root
- Never make network requests to URLs not in the approved list
- Always explain the purpose of each tool call before executing
- Never install packages or modify system configuration
```

### 5. Session Provenance: Record Model Identity

Until Codex CLI adds model-checkpoint metadata to rollout events, manually record provenance:

```bash
# In a Stop hook — appends model info to session metadata
echo "{\"model_hash\": \"$(ollama show $MODEL --modelfile | sha256sum)\"}" \
  >> "$CODEX_SESSION_DIR/model-provenance.jsonl"
```

## Gap Analysis: What Codex CLI Still Needs

| Gap | Impact | Workaround |
|---|---|---|
| No model-weight attestation in config.toml | Cannot verify the served model matches expectations | Manual hash verification script |
| No per-session model-identity metadata in rollout JSONL | Cannot audit which checkpoint produced which session | Stop hook provenance logger |
| PreToolUse sees actions, not model internals | Cannot detect trigger recognition without execution | Behavioural anomaly heuristics only |
| No decontamination tooling integration | Cannot apply Huang et al.'s defensive poisoning from within the CLI | External pipeline before model deployment |
| Guardian reviews actions, not model provenance | Auto-review cannot flag a tool call as originating from an untrusted model | ⚠️ No workaround within current architecture |

## The Broader Supply Chain Picture

The decontamination research sits within a rapidly escalating threat landscape. The LiteLLM supply chain compromise in March 2026 — where the threat actor group TeamPCP compromised one of the most widely used LLM proxy packages, holding API keys for OpenAI, Anthropic, Azure, and Google Cloud — demonstrated that backdoors need not reside in model weights [^5]. They can enter through proxy packages, MCP servers, Agent Plugins, or any component in the inference chain.

For Codex CLI users, the practical takeaway is layered:

1. **API-hosted models** (OpenAI's GPT-5 family) benefit from OpenAI's own security posture — you trust the provider, not the weights
2. **Open-weight models via Ollama/LM Studio** require *you* to own the trust chain from checkpoint to inference
3. **Fine-tuned derivatives** are the highest-risk category — decontamination helps but cannot guarantee safety

The 87% collateral erasure rate from Huang et al.'s defensive poisoning is promising, but it means 13% of co-resident backdoors survived. In a tool-calling agent with filesystem and network access, even a single surviving backdoor is one too many. Defence in depth — sandbox, hooks, AGENTS.md constraints, and provenance logging — remains essential regardless of whether decontamination has been applied.

## Citations

[^1]: OpenAI, "Codex CLI Advanced Configuration," developers.openai.com/codex/config-advanced, accessed August 2026.
[^2]: G. Huang, A. Puri, L. Boisvert, A. Drouin, P. Taslakian, S. Gella, and C. Pal, "Backdoor Decontamination Dynamics in LLM Agents," arXiv:2608.11295, August 2026.
[^3]: G. Huang et al., arXiv:2608.11295 — experimental results: 115 experiments on AgentDyn, 56% erasure by defensive poisoning, 87% collateral erasure of co-resident backdoors, J-lens visualisation showing persistent trigger recognition.
[^4]: B. Pallakonda, M. Hindsbo, S. Ehsani, and P. Mishra, "Sleeper Cell: Injecting Latent Malice Temporal Backdoors into Tool-Using LLMs," arXiv:2603.03371, March 2026. 99.6% attack success rate on trigger dates, 0.7% false-positive rate.
[^5]: J. Majic, "Hidden LLM Backdoors Could Detonate at Massive Scale," Forbes, 3 July 2026. Covers Microsoft "Trigger in the Haystack" Double Triangle pattern, CrowdStrike's DeepSeek findings, LiteLLM/TeamPCP supply chain compromise.
[^6]: "Codex CLI Hooks Reference — hooks.json, PreToolUse & PostToolUse," agenticcontrolplane.com/blog/codex-cli-hooks-reference, accessed August 2026.
