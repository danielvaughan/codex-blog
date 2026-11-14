---
title: "`codex doctor --network-only`: Fast Network Diagnostics for Proxy, MCP, and Vertex AI Issues"
date: 2026-11-05T10:00:00+00:00
last_modified_at: 2026-11-14T11:35:08Z
parent: "Articles"
nav_order: 1216
tags: ["codex-cli", "diagnostics", "troubleshooting", "networking", "proxy", "mcp", "vertex-ai", "v0.166"]
copy_edited: true
---

# `codex doctor --network-only`: Fast Network Diagnostics for Proxy, MCP, and Vertex AI Issues

![Sketchnote diagram for: `codex doctor --network-only`: Fast Network Diagnostics for Proxy, MCP, and Vertex AI Issues](/sketchnotes/articles/2026-11-05-codex-doctor-network-only-fast-network-diagnostics-proxy-mcp-vertex-ai.png)


Shipped in v0.166.0 (stable 29 October 2026), the `--network-only` flag for `codex doctor` runs a targeted subset of diagnostics — DNS, TCP/TLS, proxy detection, and OTEL collector health — without touching environment, configuration, or model checks. It is the fastest way to confirm whether a connectivity problem is real before reaching for logs or restarting infrastructure.

## Why a Separate Flag?

The full `codex doctor` suite is thorough. On a corporate network with slow DNS or high-latency proxy authentication, it can take several seconds to complete — and most of that time is spent on checks that are irrelevant when you already suspect a network problem.

`--network-only` limits the run to what matters for connectivity diagnosis:

| Check | What It Probes |
|---|---|
| **DNS resolution** | Configured endpoints: MCP servers, model providers, OTEL collectors |
| **TCP/TLS handshake** | Each resolved endpoint, with TLS certificate validation |
| **Proxy detection** | System proxy settings, PAC/WPAD resolution, authentication probe |
| **OTEL collector** | Health check if a managed OTEL config is present |

It skips: install integrity, SQLite state, terminal metadata, authentication tokens, update availability, and app-server daemon state.

## Basic Usage

```bash
# Run network-only diagnostics
codex doctor --network-only
```

Output follows the same `✓ ok`, `⚠ warn`, `✗ fail`, `○ idle` format as the full doctor suite, filtered to network checks only.

## When to Use It

**MCP server connection failures.** If a tool call fails with a timeout or connection refused, `codex doctor --network-only` confirms whether Codex CLI can reach the MCP endpoint at all. It resolves the DNS name and attempts a TCP handshake, distinguishing between DNS failure, TCP refusal, and TLS errors.

```bash
codex doctor --network-only
# Example output:
# ✓ ok     DNS: mcp.internal.example.com → 10.0.1.42
# ✗ fail   TCP: mcp.internal.example.com:443 — Connection refused
```

The check confirms the host resolves but the port is closed, narrowing the fault to firewall rules rather than DNS or proxy config.

**Proxy authentication changes.** Enterprise proxy policies rotate frequently. If Codex stops reaching the OpenAI API after a network policy change, the proxy authentication probe surfaces expired or incorrect credentials before you spend time inspecting Codex config.

**Vertex AI reachability.** For teams using GCP Vertex AI as the model provider, `--network-only` checks the Vertex AI endpoint and Workload Identity credentials used for the HTTP connection. It is the first command to run when a Vertex AI-hosted session fails to initialise.

**OTEL collector handshakes.** If you have configured managed OTEL telemetry delivery and spans are not appearing, the OTEL collector health check in `--network-only` confirms whether the collector is reachable and accepting connections before checking your telemetry pipeline.

## Combining with the Full Suite

`--network-only` is not a replacement for `codex doctor` — it is a triage filter. If the network checks pass and the problem persists, run the full suite:

```bash
# Step 1: rule out network
codex doctor --network-only

# Step 2: if network is clean, run full diagnostics
codex doctor

# Step 3: for support tickets, capture JSON output
codex doctor --json > doctor-report.json
```

For a complete reference of `codex doctor` diagnostic categories, output formats, and support-ticket workflows, see [Codex CLI Doctor: Diagnostics, Troubleshooting, and Support-Ready Reports](/2026/06/02/codex-cli-doctor-diagnostics-troubleshooting-support-workflows/).

## Version Note

`--network-only` was introduced in the v0.166.0 alpha sprint (PR #57606[^1], merged 28 October 2026) and ships in v0.166.0 stable (released 29 October 2026). It is available in all versions from v0.166.0 onwards.

---

[^1]: PR #57606 — "Add `--network-only` flag to `codex doctor`" — merged 28 October 2026 as part of the v0.166.0 alpha sprint; ships in v0.166.0 stable (29 October 2026).
