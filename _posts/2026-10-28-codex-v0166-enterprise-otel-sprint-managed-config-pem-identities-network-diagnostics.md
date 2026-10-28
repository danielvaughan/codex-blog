---
title: "Codex CLI v0.166.0 Alpha Sprint: Enterprise OpenTelemetry, PEM Client Identities, and `codex doctor --network-only`"
date: 2026-10-28T08:00:00+00:00
last_modified_at: 2026-10-28T13:04:38Z
tags:
  - codex-cli
  - observability
  - opentelemetry
  - enterprise
  - diagnostics
  - v0.166.0
copy_edited: false
premium: false
---

Five OpenTelemetry pull requests merged into the v0.166.0 alpha branch before 07:00 UTC on 28 October 2026, making the sprint theme unmistakable: enterprise-grade telemetry delivery. The v0.165.x sprint added the *generation* side — turn, streaming, and tool-dispatch tracing wired into Codex CLI's runtime. v0.166.x adds the *delivery* side: managed OTEL collector configuration, PEM-authenticated client identities, credential-safe header redaction, and trace-exporter continuity across account switches.

This article covers the v0.166.0-alpha.2 through alpha.5 changes shipped on 28 October, alongside two companion additions: a new `--network-only` flag for `codex doctor` and the removal of `codex-exec-server` from release artifacts.

## The OTEL Sprint: Five PRs, One Coherent Architecture

The five OTEL PRs form a layered enterprise telemetry stack.

### Managed OpenTelemetry Configuration Requirements (PR #57649)

The first layer is a structured configuration schema for managed OTEL collectors. Rather than requiring enterprise teams to hand-craft exporter configuration, Codex CLI now accepts a managed OTEL configuration block that expresses requirements — collector endpoint, authentication method, export interval, batch size — in a declarative form the CLI validates on startup.

This matters in regulated environments where the OTEL collector is not run by the developer but by a platform team. The CLI becomes a configuration consumer, not a configuration author.

### OpenTelemetry Requirement Types in the App-Server Protocol (PR #57638)

The managed configuration schema is surfaced through the app-server protocol — the same protocol used for MCP server authentication and credential injection. This means enterprise deployments that already use the app-server protocol for MCP auth can extend the same trust boundary to telemetry configuration without adding a separate out-of-band configuration channel.

The requirement types modelled in the protocol correspond to the fields the CLI needs to initialise its OTEL exporter: endpoint URL, authentication credentials, TLS configuration, and metadata tags.

### Header Redaction in Config Responses (PR #57654)

When the app-server returns OTEL configuration — including authorization headers for the managed collector — those headers are now redacted in any config response that Codex CLI logs, serialises, or returns to diagnostic tools. The raw credential never appears in `codex doctor` output, session logs, or telemetry payloads themselves.

This is the same pattern used for credential masking introduced in v0.164.0 (PRs #54309, #54432, #56049) for remote Bash/Zsh/custom providers. The security model is consistent: credentials injected through the app-server protocol are used but never echoed.

### Managed OpenTelemetry Metadata and PEM Client Identities (PR #57658)

The most architecturally significant addition in the sprint. Enterprise OTEL collectors increasingly use mutual TLS — the collector authenticates the client as well as the reverse. PR #57658 adds support for PEM-encoded client certificate and private key, injected through the same managed configuration path.

A PEM client identity means the Codex CLI session can be cryptographically identified at the collector. This enables:

- **Per-developer trace attribution** without relying on bearer tokens that can be copied between machines
- **Collector-side policy enforcement** — collectors can reject spans from unregistered identities
- **Audit trails tied to certificate lifecycle** — revoking a developer's certificate stops their telemetry at the collector, not just at the CLI

Configuration follows the same managed OTEL block, extended with `client_cert_pem` and `client_key_pem` fields.

### Fix Trace Exporter Activation After Account Switches (PR #57657)

The fifth OTEL PR addresses a lifecycle bug: when a user switches OpenAI accounts mid-session (or the session reconnects after an auth refresh), the trace exporter was not re-initialised with the new account context. Spans generated after the switch were either dropped or attributed to the previous account.

The fix ensures the trace exporter is paused, the managed OTEL configuration is re-fetched for the new account, and the exporter is re-activated before any spans from the new context are flushed. Trace continuity across account switches is now guaranteed.

## `codex doctor --network-only` (PR #57606)

The existing `codex doctor` command runs a full suite of diagnostic checks. For network-related issues — MCP server connectivity, proxy authentication, Vertex AI reachability, OTEL collector handshakes — the full doctor run includes checks that are irrelevant and slow.

`--network-only` limits the doctor run to network-layer checks:

- DNS resolution for configured endpoints (MCP servers, model providers, OTEL collectors)
- TCP/TLS handshake to each endpoint
- Proxy detection and authentication probe
- OTEL collector health check (if managed OTEL config is present)

The practical use is faster diagnosis in enterprise environments where the developer knows the local install is healthy but suspects a proxy policy change, firewall rule, or Vertex AI quota issue. Running `codex doctor --network-only` gives a targeted answer in seconds rather than the full multi-minute suite.

## `codex-exec-server` Removed from Release Artifacts (PR #57628)

The `codex-exec-server` binary has been removed from the release bundle. Its functionality — executing approved shell commands in the sandboxed environment — has been absorbed into the main Codex CLI runtime. The separate binary was a legacy of an earlier architecture where the exec layer was a separate process; that split no longer reflects the current runtime model.

Users who reference `codex-exec-server` directly in CI scripts or custom wrappers will need to migrate. The same command execution behaviour is available through the standard `codex` binary without any configuration change.

## Supporting Changes in Alpha.2 Through Alpha.5

Several smaller PRs landed alongside the OTEL sprint:

**Preserve rollout tracing across sub-agent reloads (PR #57615)** — when a sub-agent session is reloaded (e.g. after a compaction or reconnect), the rollout tracing context is now preserved. This prevents orphaned spans in multi-agent sessions that involve session recovery.

**Share runtime auth with model providers and catalogs (PR #57607)** — the runtime authentication token is shared with model provider and model catalog lookup requests, removing a class of auth failure where a session with valid credentials could not refresh its model catalog because the catalog fetch used a separate auth path.

**Show Bedrock setup for explicitly configured OpenAI providers (PR #57626)** — when a user has explicitly configured an OpenAI provider in `config.json` but Bedrock is available, the TUI now surfaces Bedrock setup guidance rather than silently ignoring it. Relevant for enterprise users on AWS who have an OpenAI API key but could reduce egress costs by routing through Bedrock.

**Disable terminal status sequences on older VTE backends (PR #57612)** — fixes a rendering issue on Linux terminals using older VTE backends (common in corporate Linux desktop environments) where terminal status sequences produced visible escape characters rather than progress indicators.

## What This Means for Enterprise Deployments

The v0.165.x and v0.166.x sprints taken together form a coherent enterprise observability layer:

| Layer | v0.165.x | v0.166.x |
|-------|-----------|-----------|
| Trace generation | Turn, streaming, tool-dispatch spans ✅ | Rollout tracing across reloads ✅ |
| Exporter configuration | Basic OTEL exporter setup | Managed config, PEM identities, header redaction ✅ |
| Account continuity | — | Exporter re-activation after account switch ✅ |
| Diagnostics | `codex doctor` (full suite) | `codex doctor --network-only` ✅ |

Teams running Codex CLI in corporate environments with managed OTEL collectors, mutual TLS requirements, and proxy-aware networks now have the full stack to route telemetry reliably.

## Release Timeline

| Version | Released (UTC) | Notable PRs |
|---------|----------------|-------------|
| v0.166.0-alpha.1 | Oct 27 01:33 | GPT-6.1 Terra, safety recourse TUI, session reliability |
| v0.166.0-alpha.2 | Oct 28 (early) | OTEL managed config, requirement types |
| v0.166.0-alpha.3 | Oct 28 (early) | Header redaction, PEM identities, account-switch fix |
| v0.166.0-alpha.4 | Oct 28 01:22 UTC | codex doctor --network-only, exec-server removal |
| v0.166.0-alpha.5 | Oct 28 04:43 UTC | Supporting changes (auth sharing, Bedrock setup, VTE fix) |

v0.165.0 stable shipped 27 October at 20:09 UTC. v0.166.0 stable is expected within the week given the sprint velocity.

## Related Articles

- [Codex CLI v0.165.0 Observability Sprint: Turn Tracing, Streaming Spans, and Guardian v2](../2026-10-26-v0165-observability-sprint-tracing-streaming-tool-dispatch-guardian-v2/)
- [Codex CLI v0.166.0-alpha.1: GPT-6.1 Terra, Safety Recourse TUI, and Session Reliability](../2026-10-27-codex-v0166-alpha1-gpt61-terra-safety-recourse-session-reliability/)
- [Codex CLI v0.165.0 Stable: GCP Vertex AI, GPT-6.1 Terra, and Memory v2](../2026-10-28-codex-v0165-stable-gcp-vertex-gpt61-terra-memory-v2/)
- [Codex CLI v0.164.0 Stable: /presets, Image Previews, Enterprise MCP GA, and Bedrock Default-On](../2026-10-23-codex-v0164-stable-release-presets-image-previews-enterprise-mcp-bedrock/)
