---
title: "Codex CLI Custom Model Providers: Azure, Vercel, Local LLMs and Dynamic Bearer Tokens"
layout: single
parent: "Articles"
nav_order: 131
---

# Codex CLI Custom Model Providers: Azure, Vercel, Local LLMs and Dynamic Bearer Tokens

**Date:** 2026-03-31
**Tags:** model-providers, azure, vercel, custom-provider, bearer-token, enterprise, config-toml, authentication, dynamic-refresh

Codex CLI ships wired to OpenAI's hosted models, but the `[model_providers]` configuration table lets you point it at any OpenAI-compatible endpoint — Azure AI Foundry, the Vercel AI Gateway, a self-hosted Ollama instance, or any private inference cluster your organisation runs. This article documents the full provider config surface as of v0.117.0, covers practical setup for the most common alternatives, and tracks the enterprise authentication gap that the team is actively closing.

---

## Why Custom Providers Matter

Three scenarios drive the majority of custom-provider setups:

1. **Data residency** — regulated industries (finance, healthcare, government) require code and prompts to stay in-region. Azure AI Foundry routes all traffic through your Azure subscription and never egresses to OpenAI's infrastructure.

2. **Provider redundancy / cost control** — the Vercel AI Gateway routes requests to OpenAI, Anthropic, Google, or 100+ other endpoints from a single Codex config. Teams running Codex at scale use this to A/B test models or fall back gracefully when one provider has an outage.

3. **Local models** — developers in air-gapped environments, or those who want zero-cost iteration on private code, run `gpt-5-codex` fine-tunes or other compatible models via Ollama or llama.cpp's OpenAI-compatible server.

---

## The Full Configuration Table

The `[model_providers.<id>]` table in `~/.codex/config.toml` (or `.codex/config.toml` for project-scoped overrides) accepts these fields:[^1]

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | No | Display name shown in the TUI |
| `base_url` | string | Yes | API endpoint — must be OpenAI-compatible (Responses API) |
| `env_key` | string | Recommended | Env var holding the API key. Preferred over `experimental_bearer_token` |
| `env_key_instructions` | string | No | Message shown in the TUI if the env var is missing |
| `experimental_bearer_token` | string | Discouraged | Hardcoded token — avoid in shared configs |
| `http_headers` | map | No | Static headers sent with every request |
| `env_http_headers` | map | No | Headers populated from environment variables at runtime |
| `query_params` | map | No | Query string params appended to every request |
| `requires_openai_auth` | bool | No | Set `true` only for `requires_openai_auth`-based providers. Default: `false` |
| `wire_api` | string | For non-OpenAI | Must be `"responses"` (Responses API — currently the only supported value) |
| `request_max_retries` | number | No | HTTP-level retry count. Default: 4 |
| `stream_idle_timeout_ms` | number | No | SSE stream idle timeout. Default: 300 000 ms |
| `stream_max_retries` | number | No | Retry count on stream interruptions. Default: 5 |
| `supports_websockets` | bool | No | Enable WebSocket transport for Responses API streaming |

**Activate a provider** with the top-level `model_provider = "<id>"` key. To switch providers per workflow, use profiles:

```toml
model_provider = "openai"   # default

[profiles.azure]
model_provider = "azure"
model = "gpt-5-codex"

[profiles.local]
model_provider = "local-ollama"
model = "codex-local-7b"
```

> ⚠️ `model_providers`, `mcp_servers`, and `[otel]` blocks are **not** profile-overridable — they must be defined at the top level and are available to all profiles.

---

## Azure AI Foundry

The most commonly deployed enterprise provider. Full details are in [notes/codex-azure-openai.md](/codex-resources/notes/codex-azure-openai/), but the canonical config is:

```toml
[model_providers.azure]
name = "Azure AI Foundry"
base_url = "https://<your-resource>.openai.azure.com/openai/deployments/<your-deployment>"
env_key = "AZURE_OPENAI_API_KEY"
env_key_instructions = "Set AZURE_OPENAI_API_KEY to your Azure AI Foundry key"
wire_api = "responses"
requires_openai_auth = false

[model_providers.azure.http_headers]
"api-version" = "2026-02-01"

model_provider = "azure"
model = "gpt-5-codex"          # Must match your Azure deployment name
```

**Key constraint:** Azure requires `api-version` as a query param or header. Use `http_headers` as shown — `query_params` also works (`query_params = { "api-version" = "2026-02-01" }`), but headers are cleaner with proxies.

**Entra ID limitation:** Azure AD SSO (Managed Identity / Entra tokens) is not supported as of v0.117.0. API key auth only. Watch [issue #13241](https://github.com/openai/codex/issues/13241) for updates.

---

## Vercel AI Gateway

Vercel's AI Gateway acts as a single-endpoint proxy that multiplexes across OpenAI, Anthropic, Google, and 100+ other providers. Codex treats it as one custom provider; you switch models by changing the `model` string, not the endpoint.

```toml
[model_providers.vercel]
name = "Vercel AI Gateway"
base_url = "https://ai-gateway.vercel.sh/v1"
env_key = "VERCEL_OIDC_TOKEN"    # rotates automatically via Vercel's CI
wire_api = "responses"

model_provider = "vercel"
model = "openai/gpt-5-codex"     # or "anthropic/claude-sonnet-4-5", "google/gemini-3-flash", etc.
```

Model strings follow the `<provider>/<model-id>` convention. See the Vercel AI Gateway docs for the full catalogue.

The Vercel plugin (38 skills, 3 specialist agents) is a separate install and works regardless of which provider backend you use — see [articles/2026-03-30-codex-cli-vercel-ai-gateway-skills-plugin.md](/codex-resources/articles/2026-03-30-codex-cli-vercel-ai-gateway-skills-plugin/).

---

## Local Models via Ollama

For fully offline or air-gapped setups, Ollama exposes an OpenAI-compatible endpoint at `localhost:11434`. Any model in the Ollama library can be used, subject to capability limitations (Codex's agentic loop works best with models that reliably produce tool calls in the correct schema).

```bash
# Pull a code-focused model
ollama pull qwen2.5-coder:14b
```

```toml
[model_providers.local-ollama]
name = "Ollama (local)"
base_url = "http://localhost:11434/v1"
env_key = "OLLAMA_API_KEY"      # Ollama accepts any non-empty string here
wire_api = "responses"
stream_idle_timeout_ms = 120000  # local models are slower — increase the idle timeout

model_provider = "local-ollama"
model = "qwen2.5-coder:14b"
```

**Practical caveats:**
- Context compaction's OpenAI fast path (`POST /v1/responses/compact`) is not available with non-OpenAI providers. Codex falls back to the local compaction path — a `_summary` user message. Sessions still compact, but you lose the encrypted opaque blob and pay full summarisation token cost.
- Multi-agent v2 and the `spawn_agents_on_csv` fan-out pattern work, but local models are significantly slower at following the TOML subagent protocol compared to the hosted Codex models.
- Image input/output workflows require multimodal model support — check your model's capabilities first.

---

## Bearer Token Authentication (New in v0.117.0+)

PR [#16277](https://github.com/openai/codex/pull/16277) (merged 2026-03-30) generalised the external auth token system. `ExternalAuthTokens` now carries an `access_token` plus **optional** `ExternalAuthChatgptMetadata` rather than always requiring ChatGPT account data. This means bearer-only sources — providers that issue tokens outside OpenAI's auth flow — can now use the full refresher infrastructure rather than static `experimental_bearer_token`.

**What this enables in practice:** Providers that issue access tokens via OAuth2 client credentials flow (common in enterprise Azure AD and internal IdPs) can use the `requires_openai_auth = false` path with a command-backed bearer refresher in the upcoming follow-on PR.

---

## The Dynamic Token Refresh Gap (Issue #15189)

**Current state:** Codex supports two ways to supply API keys to custom providers — `env_key` (environment variable at process start) and `experimental_bearer_token` (hardcoded). Both are static. Neither handles tokens that expire mid-session.

**Enterprise problem:** Many corporate auth systems (including Azure Managed Identity and on-prem OAuth2 IdPs) issue short-lived bearer tokens (often 5–15 minutes). A Codex session running a 45-minute agentic task will hit HTTP 401 mid-run.

**Proposed configuration** (Issue [#15189](https://github.com/openai/codex/issues/15189)):

```toml
[model_providers.enterprise-internal]
name = "Internal Inference Cluster"
base_url = "https://ai.internal.example.com/v1"
wire_api = "responses"

# Proposed — not yet shipped (watch for v0.118.x)
auth_token_command = "/usr/local/bin/generate-ai-token"
auth_token_refresh_interval_ms = 300000   # cache for 5 minutes
auth_token_refresh_on_401 = true          # force-refresh on HTTP 401
auth_token_command_timeout_ms = 5000      # fail fast if token command hangs
```

**Workaround until the feature ships:** Run a wrapper script that periodically writes a fresh token to the env var Codex reads, then launch Codex as a child process. Not elegant, but it works for CI environments where you control the wrapper.

```bash
#!/usr/bin/env bash
# token-refresh-wrapper.sh
while true; do
  export MY_PROVIDER_TOKEN=$(generate-ai-token)
  sleep 240  # refresh every 4 minutes (before the 5-minute expiry)
done &
REFRESH_PID=$!
trap "kill $REFRESH_PID" EXIT
codex "$@"
```

PR [#16277](https://github.com/openai/codex/pull/16277)'s generalised bearer auth architecture is the foundation needed to build this feature. The team described the PR as enabling "a follow-on provider-auth PR" that implements the command-backed refresher directly.

---

## Static Header Auth (API Keys as Headers)

Some providers (LiteLLM, internal proxies) use `X-API-Key` or similar custom headers rather than `Authorization: Bearer <token>`. Use `env_http_headers`:

```toml
[model_providers.litellm-proxy]
name = "LiteLLM Proxy"
base_url = "https://litellm.internal.example.com/v1"
wire_api = "responses"

[model_providers.litellm-proxy.env_http_headers]
"X-API-Key" = "LITELLM_API_KEY"   # reads from LITELLM_API_KEY env var
```

---

## Validation Checklist

Before using a custom provider in production:

```bash
# 1. Verify the provider connection
CUSTOM_API_KEY=xxx codex --profile my-provider --no-interactive \
  "Respond with 'Provider connected' and nothing else"

# 2. Confirm model availability
codex --profile my-provider model list

# 3. Test compaction (important for long sessions)
codex --profile my-provider
> /compact
# If it fails with 404, the provider doesn't support the fast path —
# that's OK, local compaction will kick in automatically

# 4. Test subagent spawn (if using multi-agent workflows)
# Ensure multi_agent feature is enabled
codex --profile my-provider features list
```

---

## What's Not Overridable Per Profile

A common mistake: defining `[model_providers]` inside a profile block. This silently fails. All provider definitions must be at the top level of `config.toml`:

```toml
# ✅ Correct — top-level provider definition
[model_providers.azure]
base_url = "..."

# ✅ Profile switches which provider to use
[profiles.azure-review]
model_provider = "azure"
model = "gpt-5-codex"
model_reasoning_effort = "high"

# ❌ Wrong — model_providers inside a profile is ignored
[profiles.azure-review.model_providers.azure]
base_url = "..."
```

---

[^1]: Source: [developers.openai.com/codex/config-reference](https://developers.openai.com/codex/config-reference) — verified 2026-03-31.
[^2]: Source: PR [#16277](https://github.com/openai/codex/pull/16277) by bolinfest, merged 2026-03-30.
[^3]: Source: GitHub Issue [#15189](https://github.com/openai/codex/issues/15189) — "Support dynamic bearer token refresh for custom model providers" — opened ~2026-03-17, open as of 2026-03-31.
