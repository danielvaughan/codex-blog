---
title: "MCP 2.0: What Claude Code 2.1.284 Actually Changed About the Protocol"
parent: "Articles"
nav_order: 1235
date: 2026-11-15T08:00:00+00:00
last_modified_at: 2026-11-15T23:05:02Z
tags:
  - mcp
  - mcp-2
  - claude-code
  - stateless-protocol
  - mcp-apps
  - enterprise
  - oauth
  - authentication
  - architecture
  - codex-cli
  - premium
copy_edited: true
---

Claude Code 2.1.284 shipped on 29 September 2026 and carried a version label for the Model Context Protocol that had not appeared before: MCP 2.0.[^1] If you have been following MCP development through the year, you might wonder what 2.0 adds on top of the 2026-07-28 specification that already eliminated sessions, rewrote the Tasks extension, and introduced MCP Apps experimentally. The answer is that Claude Code 2.1.284 is not just a client consuming the 2026-07-28 spec — it is the first production release to treat three of its capabilities as stable, documented, and ready to build against: a fully stateless core, MCP Apps as a first-class UI primitive, and Enterprise Managed Auth as a zero-configuration OAuth path for teams.[^2]

Understanding the distinction matters. The July specification defined what the protocol should do. Claude Code 2.1.284 defines what you can depend on today.

## The Stateless Core Is Now a Contract, Not a Transition

The 2026-07-28 specification removed protocol-level sessions and the `initialize`/`initialized` handshake.[^3] That was the right architectural move. What it left unresolved was the transition period: Codex CLI shipped `server/discover`-based discovery with a fallback to the legacy lifecycle for servers that had not yet migrated.[^4] The fallback existed because production servers needed time. As of Claude Code 2.1.284, the stateless core is no longer a preferred path with a compatibility bridge — it is the protocol.

What this means in practice:

**No `Mcp-Session-Id` header anywhere in the stack.** Any server-side logic that branches on session state needs to be rewritten before you can claim MCP 2.0 compliance. The header is not deprecated with a grace period in this release — it is absent.[^1]

**`Mcp-Method` and `Mcp-Name` headers are required on every POST.** These routing headers let gateways and WAFs inspect traffic without body parsing. If you are running an MCP server behind an API gateway that inspects these headers, confirm it is not rejecting requests that omit them. Claude Code 2.1.284 always sends them; older clients may not.[^3]

**`server/discover` is now the only capability negotiation path.** The `initialize` fallback that Codex CLI maintained for backward compatibility is gone. Every MCP server in a Claude Code 2.1.284 environment needs a conforming `server/discover` endpoint. Test this before the upgrade if you manage private servers.

The operational payoff is real. Stateless MCP servers run behind plain round-robin load balancers. Health checks are trivial — a 200 on `server/discover` is sufficient. Rolling restarts no longer require session drain logic. For teams that have already migrated, 2.1.284 simply confirms what they built is correct. For teams still running stateful servers behind sticky sessions, the upgrade is a forcing function.

## MCP Apps: Interactive UI Becomes a Protocol Primitive

MCP Apps were introduced experimentally in the 2026-07-28 specification as a way for servers to deliver interactive HTML interfaces rendered inside sandboxed iframes.[^3] The experiment confirmed the design. Claude Code 2.1.284 promotes MCP Apps to a stable, documented capability with defined embedding behaviour, postMessage communication, and sandbox policy that clients are expected to enforce.[^1]

The mental model is straightforward. Before MCP Apps, tools returned text, structured data, or resources. If you wanted a user to interact with a visual output — a form to approve an operation, a chart to review before proceeding, a diff rendered with syntax highlighting — you had to build that outside the MCP boundary and link to it. MCP Apps bring the interaction into the tool call response.

A server declares an MCP App result by returning a `type: "app"` content block containing a URL, a declared height, and optionally an initial data payload:

```json
{
  "type": "app",
  "url": "https://mcp.example.com/apps/approval-ui",
  "height": 480,
  "data": {
    "task_id": "deploy-prod-20261115",
    "changes": 14,
    "risk": "medium"
  }
}
```

The client renders the URL in an iframe with a sandboxed origin policy. The app communicates back to the host via `postMessage`, using a structured envelope that the protocol defines. When the user completes the interaction — clicks Approve, fills a form, selects an option — the app posts a completion event and the tool call resolves with the user's input as part of the result.[^2]

For Codex CLI users, the most immediate application is approval workflows for agentic tasks. Rather than surfacing a text prompt like `Proceed with deployment? (y/n)`, a tool can deliver a structured approval UI that shows exactly what will change, includes relevant context, and captures a structured decision. The agent waits; the developer reviews at their own pace.

There are boundaries worth noting. The sandbox policy prohibits storage access, top-level navigation, and cross-origin requests that are not explicitly permitted by the server. Apps that need to communicate with external services must route through the MCP server, not directly from the iframe. This is by design: the server remains the authority; the app is the interface.[^1]

## Enterprise Managed Auth: OAuth Without the Operational Overhead

Enterprise Managed Auth is the most significant addition for teams running Codex CLI in corporate environments.[^2] Before this release, connecting Codex CLI to an MCP server behind an enterprise identity provider required each developer to handle their own OAuth credential configuration — storing client IDs, managing token refresh, keeping secrets out of version control. At ten developers that is manageable. At a hundred it is an ongoing support burden.

Enterprise Managed Auth moves the credential configuration from the individual developer to the organisation. The OAuth client registration, the token endpoint, the scopes, and the refresh policy are defined once in the enterprise configuration and distributed to clients automatically. When a developer opens a session, the MCP authentication handshake completes without them providing anything beyond their normal SSO login.[^1]

The architecture has three components:

**The Managed Auth Registry.** The enterprise registers its MCP servers and their OAuth parameters with the identity provider. The registry maps server URLs to credential configurations. Claude Code 2.1.284 reads from this registry at session start rather than from the local `config.toml` credential stash.

**Zero-touch delegation.** When Codex CLI encounters an MCP server covered by the Managed Auth Registry, it requests a delegated token using the developer's existing SSO session. No client secret is needed locally. The token is scoped to the session and expires when the session ends.[^2]

**TTL and rotation policy enforcement.** The enterprise controls token lifetime. Short-lived tokens — fifteen minutes is a sensible default for agentic workloads — reduce the blast radius of a compromised session. The managed auth layer handles refresh automatically within the session; the developer never sees a token expiry error mid-task.[^1]

The practical effect on `config.toml` is a reduction in per-server credential blocks. Where you previously needed:

```toml
[[mcp.servers]]
name = "internal-search"
url = "https://search.internal.example.com/mcp"
auth_type = "oauth"
client_id = "..."
client_secret_env = "MCP_SEARCH_SECRET"
token_url = "https://sso.example.com/token"
```

With Enterprise Managed Auth, the entry collapses to:

```toml
[[mcp.servers]]
name = "internal-search"
url = "https://search.internal.example.com/mcp"
auth_type = "managed"
```

The managed auth layer resolves the rest. This is the zero-touch OAuth delegation the backlog item described: the developer's `config.toml` stops holding secrets and starts pointing at policy.

## What This Means for Existing MCP Configurations

If you have been following MCP migration guidance from the July and August 2026 articles in this series, your configuration is largely correct. The 2.1.284 upgrade solidifies what you already did. There are three specific checks worth running:

**Confirm your private MCP servers implement `server/discover`.** Use `codex mcp diagnose <server-name>` — available in Codex CLI v0.165+ — to verify the discovery endpoint returns a well-formed capabilities response. A legacy `initialize`-only server will fail in a 2.1.284 client environment.

**Audit MCP App sandbox compliance if you build MCP servers.** If your servers return `type: "app"` content blocks, check the returned URL for violations: storage access in the app code, top-level navigation triggers, or cross-origin fetch calls that bypass the MCP server. The stricter enforcement in 2.1.284 will silently suppress non-compliant apps rather than surfacing an error — something you will only notice by testing the interaction path.

**Migrate enterprise credential blocks to `auth_type = "managed"` once IT provisions the registry.** This is not mandatory immediately, but `auth_type = "oauth"` with local client secrets is now the legacy path. Organisations that do not migrate will find that audit tooling increasingly flags locally stored MCP credentials as a security gap.

## The Significance of the Version Number

Calling this release MCP 2.0 is a deliberate signal. The 2026-07-28 specification was the architectural work. Claude Code 2.1.284 is the moment a major agentic coding client treats that architecture as production-stable. That matters for the ecosystem: tool authors writing MCP servers can now target the stateless core and MCP Apps without hedging for clients that might still be on the legacy handshake path. The gap between specification and reliable client support, which makes protocol versioning so fraught in practice, has closed.

For Codex CLI users, the near-term implication is that any MCP integration you build from this point forward should target the 2.0 contract. Session state belongs in your server's application layer, not in the protocol. UI interactions belong in MCP Apps, not in a parallel web UI. OAuth credentials belong in managed auth policy, not in developer dotfiles.

The protocol made these things possible in July. Claude Code made them real in September. Codex CLI inherits the benefit at whatever version you are running today.

---

[^1]: Claude Code 2.1.284 release notes, Anthropic, 29 September 2026.
[^2]: Anthropic Engineering Blog, "MCP 2.0: Stable Stateless Core, MCP Apps, and Enterprise Managed Auth", 29 September 2026.
[^3]: Model Context Protocol Specification 2026-07-28, modelcontextprotocol.io, 28 July 2026.
[^4]: Codex CLI PR #35724, "MCP: server/discover-based HTTP discovery with legacy fallback", merged v0.146.0.
[^5]: "MCP OAuth Architecture in Multi-Agent Sessions", codex-resources, 12 November 2026.
