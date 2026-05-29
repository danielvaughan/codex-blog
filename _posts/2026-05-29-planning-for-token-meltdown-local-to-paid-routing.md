---
title: "Planning for Token Meltdown: How to Route Local to Paid Automatically"
description: "Local models cannot natively decide to escalate. You need a routing layer. LiteLLM running as a local proxy gives you automatic fallback from fast local models to capable cloud models based on failures, latency, or context window limits."
date: 2026-05-29T11:00:00+00:00
last_modified_at: 2026-05-29T12:40:01+01:00
layout: post
tags:
  - codex-cli
  - local-models
  - litellm
  - routing
  - ollama
  - cost-optimisation
  - production-patterns
---

# Planning for Token Meltdown: How to Route Local to Paid Automatically

![Sketchnote diagram for: Planning for Token Meltdown: How to Route Local to Paid Automatically](/sketchnotes/articles/2026-05-29-planning-for-token-meltdown-local-to-paid-routing.png)


## The problem: local models cannot escalate themselves

You run a local model for speed and cost. It handles 80 per cent of requests well. Then it hits a task that exceeds its context window, or requires reasoning it cannot produce, or simply times out under load. The request fails. You notice 20 minutes later.

Local models have no native mechanism to say 'this is beyond me, send it to something better.' They either produce an answer (possibly wrong) or fail silently. The user experience degrades without warning.

The fix is a routing layer that sits between your tools and your models, tries the fast local option first, and automatically promotes to a cloud model when the local one fails. LiteLLM[^litellm] running as a local proxy is the best current option for this.

## The architecture

```
┌─────────────┐    ┌─────────────┐    ┌──────────────┐
│  Codex CLI  │    │   Cline     │    │ Foundry      │
│  --provider │    │             │    │ Toolkit      │
└──────┬──────┘    └──────┬──────┘    └──────┬───────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  LiteLLM Proxy        │
              │  http://127.0.0.1:4000│
              │                       │
              │  Router Strategy:     │
              │  fast → smart → cloud │
              └───────┬───────┬───────┘
                      │       │
            ┌─────────┘       └─────────┐
            ▼                           ▼
   ┌─────────────────┐       ┌──────────────────┐
   │  Ollama (local)  │       │  OpenAI / Anthro  │
   │  gpt-oss:120b    │       │  gpt-5.4-mini    │
   │  qwen3:32b       │       │  claude-sonnet   │
   └──────────────────┘       └──────────────────┘
```

Every tool, Codex CLI, Cline, Foundry Toolkit custom endpoints, anything that speaks the OpenAI-compatible API, points to a single URL: `http://127.0.0.1:4000/v1`. The router handles everything else.

## Setting up LiteLLM as the routing layer

### Install and start

```bash
pip install litellm[proxy]
litellm --config config.yaml --port 4000
```

### The configuration file

This config defines three tiers: fast local, smart local, and cloud fallback. The router tries them in order.

```yaml
# config.yaml
model_list:
  # Tier 1: Fast local (Ollama)
  - model_name: fast-local
    litellm_params:
      model: ollama/qwen3:8b
      api_base: http://localhost:11434
      rpm: 120
      timeout: 30

  # Tier 2: Smart local (larger Ollama model)
  - model_name: smart-local
    litellm_params:
      model: ollama/qwen3:32b
      api_base: http://localhost:11434
      rpm: 30
      timeout: 120

  # Tier 3: Cloud fallback (OpenAI)
  - model_name: cloud-fallback
    litellm_params:
      model: openai/gpt-5.4-mini
      api_key: os.environ/OPENAI_API_KEY
      rpm: 200
      timeout: 60

  # Tier 4: Cloud heavy (for complex reasoning)
  - model_name: cloud-heavy
    litellm_params:
      model: openai/gpt-5.5
      api_key: os.environ/OPENAI_API_KEY
      rpm: 60
      timeout: 180

router_settings:
  routing_strategy: usage-based-routing
  num_retries: 2
  timeout: 30
  allowed_fails: 3
  cooldown_time: 60

litellm_settings:
  # If fast-local fails, try smart-local, then cloud
  fallbacks:
    - fast-local: ["smart-local", "cloud-fallback"]
    - smart-local: ["cloud-fallback"]
    - cloud-fallback: ["cloud-heavy"]

  # If context window exceeded, jump straight to cloud
  context_window_fallbacks:
    - fast-local: ["smart-local", "cloud-heavy"]
    - smart-local: ["cloud-heavy"]

  # Retry and timeout settings
  num_retries: 2
  request_timeout: 30
  allowed_fails: 3
  cooldown_time: 60
```

### What triggers escalation

The router promotes to the next tier when:

1. **Connection failure.** The local model is not running, has crashed, or is out of memory. LiteLLM gets a connection error and immediately tries the next tier.

2. **Timeout.** The local model takes longer than `timeout` seconds. For a fast 8B model, 30 seconds is generous. If it is not done, something is wrong.

3. **Context window exceeded.** The input exceeds the model's context limit. LiteLLM catches the specific error code and uses `context_window_fallbacks` to jump to a model with a larger window, skipping intermediate tiers that would also fail.

4. **Rate limit (429).** If you have multiple users hitting the local model and it returns 429, the router falls through.

5. **Repeated failures.** After `allowed_fails` consecutive failures, the deployment enters `cooldown_time` (60 seconds here). During cooldown, all requests route to the next tier automatically.

6. **Content policy violation.** If a local model refuses a legitimate request due to overly conservative filtering, `content_policy_fallbacks` can route to a model with different policies.

## Pointing your tools at the proxy

### Codex CLI

```toml
# ~/.codex/config.toml
[model_providers.litellm-local]
name = "LiteLLM Local Router"
base_url = "http://127.0.0.1:4000/v1"
auth = "none"

[profiles.routed]
model = "fast-local"
model_provider = "litellm-local"
```

```bash
codex --profile routed "Refactor the authentication module"
```

Codex sends the request to LiteLLM at port 4000. LiteLLM tries the fast local model first. If it fails or times out, the response comes from the cloud, and Codex never knows the difference.

### Cline

In Cline's settings, set the custom API endpoint to `http://127.0.0.1:4000/v1` and the model to `fast-local`. Cline speaks the standard OpenAI chat completions API, so it works without modification.

### Foundry Toolkit

Configure the custom endpoint in Foundry Toolkit to `http://127.0.0.1:4000/v1`. Any tool that accepts a custom OpenAI-compatible base URL works with this pattern.

### Any OpenAI-compatible tool

The proxy exposes the standard `/v1/chat/completions`, `/v1/completions`, and `/v1/embeddings` endpoints. Anything that can point at an OpenAI base URL can use this routing layer without code changes.

## Latency-based routing

The usage-based strategy above routes on failures. For a more sophisticated setup, latency-based routing promotes to cloud when the local model becomes slow under load, not just when it fails:

```yaml
router_settings:
  routing_strategy: latency-based-routing
  # Route to lowest-latency deployment
  # Local models are faster when idle, cloud is faster under load
```

With latency-based routing, the proxy measures response times across all tiers. When your local GPU is saturated and response times spike from 2 seconds to 15 seconds, the router automatically shifts traffic to the cloud tier which is responding in 3 seconds. When the local model recovers, traffic shifts back.

## Cost tracking

LiteLLM tracks spend per model and per virtual key. This is how you know the routing is working:

```yaml
litellm_settings:
  success_callback: ["langfuse"]  # or "prometheus", "datadog"

general_settings:
  master_key: sk-local-proxy-key
  database_url: postgresql://user:pass@localhost:5432/litellm
```

After a week of running, check the dashboard. You want to see:

- 70–85 per cent of requests handled locally (cost: zero)
- 10–20 per cent promoted to smart-local (cost: zero, but slower)
- 5–15 per cent falling through to cloud (cost: per-token pricing)

If cloud usage exceeds 20 per cent consistently, either your local model is too small for your workload, or your timeout thresholds are too aggressive.

## The meltdown scenario

Token meltdown happens when:

1. You start a large refactoring task that generates prompts exceeding 32,000 tokens
2. The local model's context window is 8,192 tokens (many quantised models)
3. Every request fails with a context window error
4. Without routing, you manually switch profiles or restart with a different model
5. With routing, `context_window_fallbacks` catches the error and promotes to cloud within the same request, no interruption

The second meltdown scenario is thermal: your GPU hits thermal limits under sustained load, the local model starts producing garbage or timing out, and the router quietly moves traffic to the cloud until the hardware recovers.

## What this does not solve

- **Quality routing.** LiteLLM routes on failures, timeouts and latency. It does not route on output quality. If the local model produces a wrong answer confidently, the router has no way to know.
- **Cost budgets.** You can set spend limits per key, but the router does not stop promoting to cloud when your budget is exhausted. It fails the request instead.
- **Model selection intelligence.** The complexity router[^auto-routing] adds basic task classification (routing reasoning-heavy requests to capable models), but it is not production-proven for coding workloads yet.

## The configuration in practice

For a developer running Codex CLI with a local Ollama instance on a machine with 32GB RAM and a 3090:

```yaml
model_list:
  - model_name: coding-local
    litellm_params:
      model: ollama/gpt-oss:120b
      api_base: http://localhost:11434
      timeout: 45

  - model_name: coding-cloud
    litellm_params:
      model: openai/gpt-5.4-mini
      api_key: os.environ/OPENAI_API_KEY

litellm_settings:
  fallbacks:
    - coding-local: ["coding-cloud"]
  context_window_fallbacks:
    - coding-local: ["coding-cloud"]
  num_retries: 1
  request_timeout: 45
  cooldown_time: 30
```

Two models. One local, one cloud. If local fails for any reason, cloud takes over. If the context window is too small, cloud takes over. Total configuration: 18 lines of YAML.

Point Codex CLI, Cline, and Foundry Toolkit at `http://127.0.0.1:4000/v1`. Done.

## Citations

[^litellm]: [LiteLLM: Open Source AI Gateway and Proxy, BerriAI](https://github.com/BerriAI/litellm)
[^fallbacks]: [Fallbacks and Reliability, LiteLLM Documentation](https://docs.litellm.ai/docs/proxy/reliability)
[^auto-routing]: [Auto Routing, LiteLLM Documentation](https://docs.litellm.ai/docs/proxy/auto_routing)
[^ollama-codex]: [Codex CLI Integration, Ollama Documentation](https://docs.ollama.com/integrations/codex)
[^config-ref]: [Configuration Reference, Codex CLI, OpenAI Developers](https://developers.openai.com/codex/config-reference)
[^routing]: [Routing and Load Balancing, LiteLLM Documentation](https://docs.litellm.ai/docs/routing-load-balancing)
