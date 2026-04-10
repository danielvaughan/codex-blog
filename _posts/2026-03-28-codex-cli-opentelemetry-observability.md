---
title: "Codex CLI OpenTelemetry: Observability and Metrics in Production"
date: 2026-03-28T09:00:00+00:00
tags:
  - configuration
  - observability
  - enterprise
  - config-toml
  - otel
  - opentelemetry
  - metrics
  - tracing
---
![Sketchnote diagram for: Codex CLI OpenTelemetry: Observability and Metrics in Production](/sketchnotes/articles/2026-03-28-codex-cli-opentelemetry-observability.png)

*Codex CLI ships built-in OpenTelemetry support for production observability — traces, logs, and metrics from every agent run, exportable to any OTLP-compatible backend.*

---

## Why Observability Matters for Agentic Workflows

Running Codex in a single session is debuggable. Running it across an agentic pod — orchestrator, planner, multiple workers, reviewers — generates complexity that escapes manual inspection. You need:

- **Latency visibility:** Which agent is taking longest? Is it the model or the tool calls?
- **Cost tracking:** How many tokens per session, per agent type, per day?
- **Failure analysis:** Where in a multi-agent chain do things go wrong?
- **Audit trails:** What did the agent do, in what order, with what approvals?

Codex CLI's `[otel]` section in `config.toml` enables all of this through standard OpenTelemetry.

---

## The `[otel]` Configuration Table

OTel export is **opt-in** and **disabled by default** (local runs remain self-contained). Enable it by adding an `[otel]` section to `~/.codex/config.toml` or your project `.codex/config.toml`.

### Minimal setup (capture but don't export):

```toml
[otel]
environment = "dev"       # tags events with env=dev
exporter = "none"         # records locally, sends nothing
log_user_prompt = false   # redacts prompt text (default)
```

### Export via OTLP/HTTP:

```toml
[otel]
environment = "staging"
exporter = { otlp-http = { endpoint = "https://otel.example.com/v1/logs", protocol = "binary", headers = { "x-otlp-api-key" = "${OTLP_TOKEN}" } } }
log_user_prompt = false
```

### Export via OTLP/gRPC:

```toml
[otel]
environment = "production"
exporter = { otlp-grpc = { endpoint = "https://otel.example.com:4317", headers = { "x-otlp-meta" = "${OTLP_META}" } } }
```

**TLS support:** The config reference supports full mutual TLS:

```toml
[otel]
tls_ca_cert = "/etc/pki/otel-ca.pem"
tls_client_cert = "/etc/pki/client.pem"
tls_client_key = "/etc/pki/client.key"
```

---

## What Gets Exported

Codex tags every exported event with:
- `service.name = "codex-cli"`
- `app.version` — CLI version
- `env` — your configured environment
- `auth_mode`, `originator`, `session_source`, `model`

**Spans and logs** cover:
- Outbound API requests and streamed responses
- User input (if `log_user_prompt = true`)
- Tool-approval decisions
- Results of every tool invocation (Bash, FileRead, FileWrite, etc.)

**Metrics** (counters + duration histograms):
- API call counts and latencies
- Stream throughput
- Tool call volumes per type
- Token counts per session

Exporters **batch asynchronously** and flush on shutdown — no blocking during agent runs.

---

## Known Gaps

Two Codex entry points have incomplete OTel support (as of v0.117.0):

| Entry Point | Traces | Logs | Metrics |
|------------|--------|------|---------|
| `codex` (interactive) | ✅ | ✅ | ✅ |
| `codex exec` | ✅ | ✅ | ❌ no metrics |
| `codex mcp-server` | ❌ | ❌ | ❌ |

**Tracked:** [Issue #12913](https://github.com/openai/codex/issues/12913) — `codex exec` and `codex mcp-server` observability gaps.

There is also a known bug where plain HTTP was observed being sent to a gRPC endpoint specified with `https://` — related to an upstream `opentelemetry-rust` issue. Use `otlp-http` with a binary protocol endpoint if you hit TLS/protocol mismatches.

---

## Integrating with Observability Backends

### SigNoz

Point the OTLP endpoint at your SigNoz Cloud ingest URL. Traces appear automatically in the Traces tab under `service.name = codex-cli`. SigNoz's [Codex monitoring guide](https://signoz.io/docs/codex-monitoring/) covers the full setup.

### Grafana + Jaeger

Run an OTel Collector as middleware:

```yaml
# otel-collector.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: "0.0.0.0:4317"

processors:
  batch: {}
  # optionally filter sensitive prompts here

exporters:
  jaeger:
    endpoint: "jaeger:14250"
  prometheus:
    endpoint: "0.0.0.0:8889"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [jaeger]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
```

```toml
# ~/.codex/config.toml
[otel]
environment = "production"
exporter = { otlp-grpc = { endpoint = "http://localhost:4317" } }
```

**Collector advantages:**
- Redact sensitive fields before they leave your network
- Batch and retry on network failures
- Add Kubernetes pod/namespace context automatically
- Fan out to multiple backends from one agent config

### Opik (Comet)

Opik maps Codex OTel settings to its own ingest endpoints — useful if you're already tracking ML experiments in Comet. See the [Opik Codex integration guide](https://www.comet.com/docs/opik/integrations/openai-codex).

### VictoriaMetrics

For a fully open-source cost-effective stack, VictoriaMetrics + OTel Collector gives you deep insights into Codex token economics and tool call volumes. The VictoriaMetrics [vibe coding observability guide](https://victoriametrics.com/blog/vibe-coding-observability/) covers all major coding agents including Codex.

---

## Building a Cost Dashboard

Token metrics from OTel give you the raw data for a cost dashboard. The key metric to track: `tokens.used` per session, segmented by `model`.

Pair with the `postTaskComplete` hook to log a cost record to `~/.codex/usage.jsonl` at session end:

```json
// ~/.codex/config.toml - hooks section
[hooks]
postTaskComplete = "~/.codex/hooks/log-cost.sh"
```

```bash
#!/bin/bash
# ~/.codex/hooks/log-cost.sh
# Called with session metadata on stdin as JSON
jq -c '{ts: now, model: .model, tokens: .token_count, session: .session_id}' >> ~/.codex/usage.jsonl
```

Then in Grafana: count tokens by model, multiply by model price, sum per day. You get a live cost graph without any third-party tracking.

---

## Privacy

**Rule:** Keep `log_user_prompt = false` (the default) unless your policy explicitly permits prompt text export.

Even with `log_user_prompt = false`, tool call results are exported — which can contain file contents if the agent read sensitive files. Use the OTel Collector processor to redact or filter before forwarding to external backends:

```yaml
processors:
  redaction:
    allow_all_keys: false
    blocked_values: ["AKIA.*", "sk-.*", "ghp_.*"]  # AWS/OpenAI/GitHub keys
```

---

## Per-Profile OTel Tuning

You can vary OTel verbosity by profile:

```toml
[profiles.ci]
model = "gpt-5-codex"
# CI: full telemetry to production collector
[otel]
environment = "ci"
exporter = { otlp-http = { endpoint = "${CI_OTEL_ENDPOINT}", headers = { "Authorization" = "Bearer ${CI_OTEL_TOKEN}" } } }

[profiles.dev]
model = "gpt-5-codex-mini"
# Local dev: capture but don't export
[otel]
environment = "dev"
exporter = "none"
```

---

## Summary

| Use case | Config |
|----------|--------|
| Local development | `exporter = "none"` — records locally, no network |
| Staging / CI | OTLP/HTTP to your collector with `environment = "staging"` |
| Production | OTLP/gRPC + full TLS, OTel Collector for redaction |
| Enterprise audit trail | `log_user_prompt = true` (if policy allows) + OTel Collector |

**Key gap to watch:** `codex exec` and `codex mcp-server` don't emit metrics yet — track [Issue #12913](https://github.com/openai/codex/issues/12913) for when this is fixed.

---

*Published: 2026-03-28*
*Sources: [Official advanced config](https://developers.openai.com/codex/config-advanced) · [SigNoz Codex docs](https://signoz.io/docs/codex-monitoring/) · [Issue #12913](https://github.com/openai/codex/issues/12913) · [VictoriaMetrics blog](https://victoriametrics.com/blog/vibe-coding-observability/) · [Opik integration](https://www.comet.com/docs/opik/integrations/openai-codex)*
