---
title: "Securing Codex CLI: Domain Allowlists, Bearer Tokens, and Network Policy Enforcement"
layout: default
parent: "Articles"
nav_order: 119
---

# Securing Codex CLI: Domain Allowlists, Bearer Tokens, and Network Policy Enforcement

**Date:** 2026-03-30
**Tags:** network-security, domain-allowlist, bearer-token, socks5, sandbox, managed-proxy, approvals, enterprise

Giving an AI agent internet access is a force multiplier — and a significant attack surface. Codex CLI's sandbox architecture treats network access as a privilege to be granted deliberately, not a default capability to be restricted after the fact. This article covers the full layered security model: from the managed proxy's domain allowlist and HTTP method filtering, through SOCKS5 proxy integration, to bearer-token authentication on remote app-server WebSocket connections introduced in v0.117.0.[^1]

---

## The Default Stance: Network Off

By design, every Codex sandbox mode starts with network access disabled for the agent phase.[^2] This matters because it is the *agent phase* — where the model executes shell commands and reads files — where prompt injection and data exfiltration risks are highest. Setup scripts that run before the agent phase retain internet access so that dependencies can be installed normally.

The three sandbox modes and their defaults:

| Mode | Network | File writes | Intended use |
|------|---------|-------------|--------------|
| `read-only` | Off | None | Safe exploration |
| `workspace-write` | Off (configurable) | Workspace only | Standard development |
| `danger-full-access` | On | Unrestricted | High-risk / explicitly opted-in |

In `danger-full-access`, the `web_search` mode also automatically switches from `"cached"` (OpenAI's pre-indexed content) to `"live"` (real-time fetching).[^3] That distinction matters for prompt injection: cached search results are curated by OpenAI, whereas live mode can retrieve arbitrary third-party content that could contain injected instructions.

---

## The Managed Proxy Layer

When you enable internet access, traffic does not flow freely. Codex routes subprocess-level outbound connections through a **managed proxy** that enforces domain and method policies.[^4] The proxy runs locally alongside the agent; it is not a cloud service.

The `[permissions.network]` table in `config.toml` governs the managed proxy:

```toml
[permissions.network]
enabled      = true
mode         = "limited"             # limited | full
proxy_url    = "http://127.0.0.1:43128"
admin_url    = "http://127.0.0.1:43129"

# Domain filtering
allowed_domains = ["registry.npmjs.org", "pypi.org", "api.github.com"]
denied_domains  = ["example.com"]

# SOCKS5 listener (for tools that speak SOCKS5 natively)
enable_socks5     = false
socks_url         = "http://127.0.0.1:43130"
enable_socks5_udp = false

# Advanced / dangerous overrides
allow_upstream_proxy                    = false
dangerously_allow_non_loopback_proxy    = false
dangerously_allow_non_loopback_admin    = false
dangerously_allow_all_unix_sockets      = false
```

The `mode = "limited"` setting (the default) enforces domain/method policies strictly. `mode = "full"` passes all traffic through — effectively disabling domain filtering while keeping the proxy in place for logging purposes.

### Domain Allowlists

`allowed_domains` takes an array of strings. As of v0.117.0, **wildcard matching** is supported.[^5] This lets you permit an entire TLD or subdomain tree without enumerating every host:

```toml
allowed_domains = [
  "*.npmjs.org",
  "*.pypi.org",
  "*.github.com",
  "*.githubusercontent.com",
  "api.openai.com",
]
```

OpenAI provides three preset profiles in Codex web environments:[^6]

- **None** — empty list; you enumerate every permitted domain.
- **Common dependencies** — a curated set of 70+ package registries, CDNs, and source control hosts (npm, PyPI, crates.io, Docker Hub, Maven, etc.).
- **All (unrestricted)** — no filtering.

The CLI equivalent is to populate `allowed_domains` manually or reference a shared `managed_config.toml` for team-wide defaults.

### HTTP Method Filtering

Domain allowlisting answers *where* the agent can connect; HTTP method filtering answers *what it can do* once connected. With filtering active, only idempotent, read-only methods pass:

- **Permitted:** `GET`, `HEAD`, `OPTIONS`
- **Blocked:** `POST`, `PUT`, `PATCH`, `DELETE` and all others

This prevents the agent from submitting forms, calling state-changing API endpoints, or exfiltrating data via a POST body — even to an explicitly allowlisted domain.[^7]

---

## SOCKS5 Proxy Integration

Codex has supported SOCKS5 proxies since v0.93.0 (December 2025), primarily for corporate environments that route all outbound traffic through a SOCKS5 gateway.[^8]

### CLI outbound traffic

For the CLI process itself, configure the proxy under `[network]`:

```toml
[network]
proxy    = "socks5://proxy.corp.example.com:1080"
no_proxy = ["localhost", "127.0.0.1", "*.internal.example.com"]
```

Authenticated proxies use the standard `user:pass@host:port` form:

```toml
proxy = "socks5://user:s3cr3t@proxy.corp.example.com:1080"
```

Standard environment variables (`ALL_PROXY`, `HTTPS_PROXY`, `HTTP_PROXY`) are also honoured automatically. This means most CI environments that already set these variables will route Codex traffic through the corporate proxy without any config change.

### Exposing a SOCKS5 listener for agent subprocesses

For tools running inside the sandbox that speak SOCKS5 natively (e.g. Git with `socks5://` in `http.proxy`), you can expose the managed proxy as a SOCKS5 listener:

```toml
[permissions.network]
enable_socks5 = true
socks_url     = "http://127.0.0.1:43130"
```

UDP-over-SOCKS5 is disabled by default (`enable_socks5_udp = false`) and should only be enabled if a specific tool requires it, as it widens the attack surface.

### Chaining to an upstream corporate proxy

If your organisation's SOCKS5 gateway sits upstream of the managed proxy:

```toml
[permissions.network]
allow_upstream_proxy = true
```

This allows the managed proxy to forward filtered traffic to the corporate proxy rather than connecting directly — preserving domain allowlist enforcement while respecting corporate routing requirements.

---

## Bearer Token Authentication for Remote App-Server Connections

v0.117.0 (26 March 2026) introduced two bearer-token authentication modes for the app-server's WebSocket listener.[^9] This is primarily relevant when Codex is embedded in a remote environment (CI runner, cloud VM) and a client UI needs to connect over a network rather than loopback.

```mermaid
sequenceDiagram
    participant Client as Remote Client
    participant WS as App-Server WebSocket
    participant Auth as Auth Middleware
    participant JSONRPC as JSON-RPC Handler

    Client->>WS: WebSocket upgrade + Authorization: Bearer <token>
    WS->>Auth: Validate token (pre-init)
    alt Token invalid
        Auth-->>Client: HTTP 401 Unauthorized
    else Token valid
        Auth->>JSONRPC: Pass connection
        Client->>JSONRPC: {"method": "initialize", ...}
        JSONRPC-->>Client: Initialized
    end
```

### Mode 1: Capability token

A simple static token stored in a file. Low overhead; suitable for single-client deployments or SSH port-forwarding scenarios.

```bash
codex app-server \
  --ws-auth capability-token \
  --ws-token-file /run/secrets/codex-token
```

The client presents it as a standard Bearer token in the HTTP upgrade request:

```http
Authorization: Bearer <token-file-contents>
```

### Mode 2: Signed bearer token (HMAC-signed JWS)

For environments with multiple clients or token rotation requirements, signed tokens provide time-bounded credentials verifiable without a database lookup.

```bash
codex app-server \
  --ws-auth signed-bearer-token \
  --ws-shared-secret-file /run/secrets/codex-hmac-secret \
  --ws-issuer "ci-runner" \
  --ws-audience "codex-agent" \
  --ws-max-clock-skew-seconds 30
```

Clock skew tolerance (`--ws-max-clock-skew-seconds`) is critical in CI environments where container clocks may drift. Tokens are validated against `iss` and `aud` claims; mismatches are rejected before any JSON-RPC processing occurs.

**Important:** The loopback address (`ws://127.0.0.1`) is the recommended listener for SSH port-forwarding deployments. Non-loopback listeners currently allow unauthenticated connections by default during rollout.[^10] If you expose the WebSocket on `0.0.0.0`, use bearer-token auth and firewall the port externally.

### Handling auth errors

Authentication is enforced *before* the JSON-RPC `initialize` call. Two error conditions to handle gracefully in clients:

- Pre-init requests return: `"Not initialized"`
- Server overload returns JSON-RPC error code `-32001`: `"Server overloaded; retry later"` — implement exponential backoff with jitter.

---

## Approval Policies

Sandbox permissions control what Codex *can* do; approval policies control when it *must ask first*. The two are orthogonal and should both be configured.[^11]

```toml
# Coarse-grained options:
approval_policy = "on-request"  # prompt for network access and risky ops
approval_policy = "never"       # full autonomy; use only in highly controlled envs
approval_policy = "untrusted"   # auto-allow known-safe ops; prompt for risky

# Fine-grained control:
approval_policy = { granular = {
  sandbox_approval    = true,
  rules               = true,
  mcp_elicitations    = true,
  request_permissions = false,
  skill_approval      = false
} }
```

Use `/approvals` or `/permissions` in the interactive CLI to inspect and change the active policy at runtime without restarting the session.

---

## Enterprise: Managed Configuration

For MDM-managed deployments (macOS via Jamf Pro, Fleet, Kandji), two preference keys under the `com.openai.codex` domain control policy enforcement:[^12]

- **`config_toml_base64`** — encoded defaults that apply unless overridden by project config.
- **`requirements_toml_base64`** — encoded constraints that Codex *rejects* any user config conflicting with.

Configuration precedence (highest to lowest): MDM managed preferences → `managed_config.toml` → `~/.codex/config.toml` → project `.codex/config.toml`.

A minimal `requirements.toml` to enforce safe web-search only across the fleet:

```toml
allowed_web_search_modes = ["cached"]
```

Combined with a `managed_config.toml` that defaults network off:

```toml
[sandbox_workspace_write]
network_access = false
```

This ensures that even if a developer sets `network_access = true` in their project config, the managed requirement takes precedence.

---

## Threat Model Summary

OpenAI explicitly documents four categories of risk when internet access is enabled:[^13]

1. **Prompt injection** — Agent fetches a web page or README containing crafted instructions that alter its behaviour. Mitigate with `web_search = "cached"` and tight domain allowlists.
2. **Data exfiltration** — Malicious scripts pipe environment variables, secrets, or git history to attacker-controlled endpoints. Mitigate with HTTP method filtering (block POST) and denylist of exfiltration targets.
3. **Malware inclusion** — Typosquatted packages or compromised dependencies fetched at install time. Mitigate with checksum verification in setup scripts and pinned dependency versions.
4. **Licence contamination** — Copyleft content incorporated without review. Mitigate with post-session licence scanning.

The layered approach — sandbox off by default, managed proxy with domain allowlist, method filtering, and approval prompts — addresses all four threat categories at different points in the attack chain.

---

## Citations

[^1]: [Codex CLI Changelog – v0.117.0, 2026-03-26](https://developers.openai.com/codex/changelog) — App-server bearer-token WebSocket auth and global network allowlist wildcard.
[^2]: [Agent internet access – Codex web | OpenAI Developers](https://developers.openai.com/codex/cloud/internet-access) — "By default, the agent runs with network access turned off."
[^3]: [Configuration Reference – Codex | OpenAI Developers](https://developers.openai.com/codex/config-reference) — `web_search` field; `danger-full-access` defaults to `"live"`.
[^4]: [Sample Configuration – Codex | OpenAI Developers](https://developers.openai.com/codex/config-sample) — Full `[permissions.network]` block with managed proxy fields.
[^5]: [Codex CLI Changelog – v0.117.0, 2026-03-26](https://developers.openai.com/codex/changelog) — "Allow global network allowlist wildcard" (PR #15549).
[^6]: [Agent internet access – Codex web | OpenAI Developers](https://developers.openai.com/codex/cloud/internet-access) — Three preset domain profiles: None, Common dependencies, All.
[^7]: [Agent approvals & security – Codex | OpenAI Developers](https://developers.openai.com/codex/agent-approvals-security) — HTTP method filtering allowing only GET/HEAD/OPTIONS.
[^8]: [Codex CLI Changelog – v0.93.0, ~Dec 2025](https://developers.openai.com/codex/changelog) — SOCKS5 proxy support with policy enforcement.
[^9]: [Codex App-Server README – GitHub openai/codex](https://github.com/openai/codex/blob/main/codex-rs/app-server/README.md) — `--ws-auth` modes, `--ws-token-file`, `--ws-shared-secret-file` flags.
[^10]: [Codex App-Server README – GitHub openai/codex](https://github.com/openai/codex/blob/main/codex-rs/app-server/README.md) — Non-loopback listeners and authentication caveats.
[^11]: [Agent approvals & security – Codex | OpenAI Developers](https://developers.openai.com/codex/agent-approvals-security) — Approval policy configuration and runtime `/approvals` command.
[^12]: [Managed configuration – Codex | OpenAI Developers](https://developers.openai.com/codex/enterprise/managed-configuration) — MDM preference keys and configuration precedence.
[^13]: [Agent internet access – Codex web | OpenAI Developers](https://developers.openai.com/codex/cloud/internet-access) — Security risks section: prompt injection, exfiltration, malware, licence contamination.
