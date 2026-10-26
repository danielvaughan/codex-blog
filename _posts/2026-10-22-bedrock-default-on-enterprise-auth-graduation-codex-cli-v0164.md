---
title: "Bedrock Default-On: How v0.164.0's Alpha.9–11 Sprint Completed the Enterprise Auth Architecture"
date: 2026-10-22T13:00:00+00:00
last_modified_at: 2026-10-26T07:16:39Z
parent: "Articles"
nav_order: 1197
tags:
  - codex-cli
  - v0.164.0
  - aws
  - bedrock
  - enterprise
  - authentication
  - credential-management
  - oauth
copy_edited: true
---

# Bedrock Default-On: How v0.164.0's Alpha.9–11 Sprint Completed the Enterprise Auth Architecture

![Sketchnote diagram for: Bedrock Default-On: How v0.164.0's Alpha.9–11 Sprint Completed the Enterprise Auth Architecture](/sketchnotes/articles/2026-10-22-bedrock-default-on-enterprise-auth-graduation-codex-cli-v0164.png)

**Release track:** v0.164.0-alpha.9 (21 October 2026), alpha.10 (22 October 2026, 00:04 UTC), alpha.11 (22 October 2026, 04:22 UTC)
**Stable track:** v0.163.0 (current stable, 15 October 2026)
**Impact:** AWS enterprise deployments, credential caching, remote execution auth, MCP enterprise auth fixes

---

Three alphas shipped in fourteen hours — alpha.9 at 14:12 UTC on 21 October, alpha.10 at 00:04 UTC on 22 October, alpha.11 at 04:22 UTC on 22 October. Taken together, they complete a theme that has been building across the entire v0.164.0 alpha sprint: graduating the enterprise authentication architecture from experimental to production-ready.

The headline change is the simplest to describe: the AWS Bedrock setup wizard is now enabled by default. But the headline understates what actually landed.

---

## The Headline: Bedrock Is Now a First-Class Setup Option

PR #56033, authored by alexsong-oai and merged at 00:50 UTC on 22 October, enables the Bedrock setup wizard by default. PR #56023, landing alongside it, groups Bedrock API key entry with other AWS sign-in methods in the credential UI.

Before this change, a developer setting up Codex CLI for the first time would see the standard OpenAI authentication flow. Bedrock was available — it has been for months — but reaching it required either knowing to look for it or being guided there by documentation. The wizard gate meant Bedrock access was effectively opt-in even for organisations that had already standardised on AWS.

With the wizard default-on, first-run setup now presents Bedrock as a first-class choice alongside the standard OpenAI credential path. For enterprise procurement teams that have approved AWS Bedrock but not direct OpenAI API access, this removes a setup friction point that has been a recurring enterprise support topic. For developers already on AWS, it means the natural path through onboarding takes them where they want to go without a detour through documentation.

The credential grouping change in #56023 is equally important practically. Bedrock API key entry now sits alongside other AWS sign-in methods in the UI — cross-account role assumption, AWS SSO, IAM credential files — rather than in a separate section. For teams using AWS IAM Identity Centre or organisation-level SSO, this means credential entry follows a familiar AWS pattern rather than a Codex-specific one.

---

## The Architecture Layer: Credential Caching and Request-Scoped OAuth

While the Bedrock wizard is user-visible, the more significant changes in alpha.10 and alpha.11 operate below the surface of the UI.

### Cache Decrypted Codex Auth and Managed Secrets (PR #56081, Celia Chen)

Every time Codex makes an authenticated API call, it needs access to the decrypted form of the stored credential. Before this PR, that decryption happened on every call. In practice, for most users, decryption is fast enough to be invisible. For enterprise deployments — where credentials may be stored in hardware security modules, where the decryption path may involve a network call to a secrets vault, or where session initialisation involves multiple credential types simultaneously — decryption on every call adds measurable latency.

PR #56081 introduces a request-scoped credential cache: decrypted credentials are cached for the duration of a request (or a bounded window) and reused across the tool calls that make up that request. The cold-start improvement is most visible in complex agentic workflows where a single user-facing action triggers dozens of downstream tool calls, each of which previously triggered its own credential decryption.

This is the performance payoff for the auth overhaul work done in alpha.7 and alpha.8. The request-scoped OAuth credentials introduced in PR #56036 (smurching-oai) formalise the credential lifecycle that the cache depends on: credentials are now explicitly scoped to a session, with defined acquisition, use, refresh, and expiry semantics that the cache can rely on.

### Require Managed Configuration from the Configured Server (PR #56055, smurching-oai)

Enterprise deployments typically use a managed configuration server — a central source of truth for which models, tools, and policies are available to a given user or team. Before this PR, Codex would fall back to local defaults if the managed configuration server was unreachable, silently degrading to an unmanaged state.

PR #56055 makes managed configuration required when a configuration server is specified. If the server is unreachable, Codex fails fast rather than silently falling back. For security and compliance teams, this is the correct behaviour: an agent operating with a stale or absent managed configuration is an agent operating outside its governance envelope.

### Support Configured Credential Providers in Remote Execution (PR #56049, viyatb-oai)

Remote execution — tasks that run on a connected host after the local session ends — previously could not use the full credential provider stack available to interactive sessions. PR #56049 extends credential provider support to remote execution contexts, so that tasks delegated to a remote host can authenticate using the same provider chain (AWS credential files, environment variables, instance profiles, SSO) as the local session that spawned them.

This closes a practical gap that has prevented some enterprise remote execution workflows from working cleanly: tasks would start successfully on the local session but fail during remote execution when they encountered a credential provider that was not supported in that context.

### Scope MITM Credential Substitution to Trusted Origins (PR #56003, viyatb-oai)

The credential substitution mechanism — which allows Codex to intercept and rewrite credentials in outbound requests — has been narrowed to apply only to trusted origins and placeholder values. Previously it applied more broadly, creating a potential for inadvertent credential rewriting when Codex was deployed in environments with complex proxy topologies.

The change is a security hardening measure. It does not affect normal operation for most users. For enterprise security teams auditing the credential flow, it closes a potential over-application of credential substitution that could theoretically have affected non-Codex traffic routed through a shared proxy.

---

## Enterprise MCP Auth: The alpha.9 Fixes

Alpha.9 also brought four enterprise MCP auth changes from jif-oai, all targeting the enterprise MCP credential selection and reauthentication path:

**PR #55859 — Fix enterprise MCP credential selection and reauthentication.** The substantive fix: corrects credential selection logic when multiple enterprise MCP configurations are active simultaneously, and fixes a reauthentication path that could leave the session in an inconsistent state after a token rotation.

**PR #55883 — Expand enterprise MCP authorisation lifecycle test coverage.** Test coverage for the credential selection fix. Adds lifecycle tests that exercise token acquisition, rotation, expiry, and reauthentication across the enterprise MCP auth state machine.

**PR #55856 — Bound managed app-server and updater stderr logging.** Operational tidying: prevents log flooding from the managed app-server and updater in high-frequency credential rotation environments.

**PR #55847 — Bound retained turn analytics and clean up closed threads.** Memory management fix for long-running sessions with high tool call volume — retained turn analytics were accumulating without bound.

---

## What This Sprint Means for v0.164.0 Stable Candidacy

The alpha.9–11 batch completes the enterprise auth architecture that has been the dominant theme of the v0.164.0 alpha sprint:

- **alpha.6** (Oct 19): Observation hooks, zstd compression, catalogue cache
- **alpha.7–8** (Oct 20–21): MCP OAuth default-on, enterprise MCP account grant reuse, zero-trust agent runtime
- **alpha.9–11** (Oct 21–22): Bedrock default-on, credential caching, remote execution providers, managed config enforcement, MCP auth fixes

The pattern is a layered construction: first the protocol changes (OAuth refresh, zero-trust registration), then the performance improvements that depend on those protocols (credential caching, request-scoped lifecycle), then the enterprise integration layer (Bedrock first-class, managed config enforcement, remote execution parity). With that stack complete, the remaining alpha work is likely to be stability, test coverage, and polish rather than new architectural additions.

The v0.164.0-alpha.11.1 backport track running in parallel on v0.163.0 — which targets the macOS `-32603` bootstrap blocker and Windows/Ubuntu TUI regressions — suggests the team is simultaneously preparing a v0.163.1 stable patch. These two tracks converging is a signal that v0.164.0 stable candidacy is approaching.

---

## Practical Notes for Enterprise Teams

**If you are evaluating Codex CLI on AWS Bedrock:** First-run setup now presents Bedrock as a default option. The wizard path through Bedrock API key entry and AWS sign-in grouping follows standard AWS credential patterns. No flags or undocumented configuration required.

**If you use a managed configuration server:** Update your deployment to treat configuration server reachability as a hard requirement (`PR #56055`). Silence previously masked misconfiguration. If the configuration server is unavailable, the correct outcome is a clear failure, not a degraded session.

**If you run remote execution workflows:** The expanded credential provider support in PR #56049 means you can now use the same provider chain (IAM profiles, SSO, credential files) in remote execution contexts as in interactive sessions. Workflows that previously failed during remote execution due to provider limitations should now work.

**If you operate long-running agentic tasks:** Credential caching (PR #56081) and the request-scoped OAuth lifecycle (PR #56036) together reduce auth-related latency in high-frequency tool call workflows. No configuration change required — the cache is on by default.

---

## Summary

The alpha.9–11 sprint is the completion of v0.164.0's enterprise auth architecture. Bedrock default-on is the most visible signal of that completion — it marks the point at which AWS is treated as a first-class peer to the standard OpenAI credential path, not an opt-in addition. The credential caching, managed config enforcement, and remote execution provider support are the structural underpinnings that make enterprise deployment reliable enough to warrant that default-on status.

v0.164.0 has not yet gone stable. But with this sprint complete, the authentication foundation it will ship on is substantially more robust than the one v0.163.0 launched with.

---

*Sources: Codex CLI GitHub repository — PRs #56033, #56023, #56081, #56055, #56049, #56036, #56003, #55859, #55883, #55856, #55847. Release dates from GitHub API.*
