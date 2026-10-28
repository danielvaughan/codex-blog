---
title: "Codex CLI v0.165.0 Stable: GCP Vertex AI Built-In, GPT-6.1 Terra, Memory v2 Default"
date: 2026-10-28T07:00:00+00:00
last_modified_at: 2026-10-28T11:36:16Z
parent: "Articles"
nav_order: 1208
tags:
  - codex-cli
  - v0.165.0
  - release-notes
  - gcp-vertex-ai
  - gpt-6-1-terra
  - memory-v2
  - observability
  - enterprise
copy_edited: true
---

# Codex CLI v0.165.0 Stable: GCP Vertex AI Built-In, GPT-6.1 Terra, Memory v2 Default

![Sketchnote diagram for: Codex CLI v0.165.0 Stable: GCP Vertex AI Built-In, GPT-6.1 Terra, Memory v2 Default](/sketchnotes/articles/2026-10-28-codex-v0165-stable-gcp-vertex-gpt61-terra-memory-v2.png)

**Release date:** 27 October 2026, 20:09 UTC
**Version:** v0.165.0 stable
**Previous stable:** v0.164.0 (23 October 2026, 04:48 UTC)
**Sprint duration:** 4 days, 13 hours (alpha.1 to stable)

---

Codex CLI v0.165.0 reached stable on 27 October 2026, four days after v0.164.0 shipped. The release completes the observability sprint that opened on 23 October: turn-level tracing, streaming instrumentation, tool-dispatch logging, and Guardian v2 conversation mode are now generally available. Alongside the observability graduation, v0.165.0 delivers three significant changes that affect every user — GCP Vertex AI as a built-in provider, GPT-6.1 Terra in the model catalogue, and Memory v2 on by default for new installs.

For a detailed look at what the observability sprint built, see [Inside the Black Box: Codex CLI v0.165.0's Observability Sprint](../2026-10-26-v0165-observability-sprint-tracing-streaming-tool-dispatch-guardian-v2/) and [GCP Vertex AI Provider: How v0.165.0-alpha.1 Expands Codex Beyond OpenAI's Infrastructure](../2026-10-23-gcp-vertex-ai-provider-codex-cli-alpha11-1/).

---

## What Graduated to Stable

### Built-In GCP Vertex AI Provider

The most operationally significant change in v0.165.0 is the native Google Cloud Vertex AI integration. Previous versions required custom provider configuration or external proxy layers to route Codex traffic to Vertex AI. v0.165.0 makes GCP a first-class provider with a single configuration option.

Configure it in `~/.codex/config.json`:

```json
{
  "model_provider": "gcp",
  "gcp_project": "my-project-id",
  "gcp_location": "us-central1"
}
```

Once set, Codex CLI communicates directly with Vertex AI without any additional setup. The integration supports the full Codex feature set including Guardian, multi-agent orchestration, MCP tool dispatch, and session resumption.

This matters for three categories of user:

**Enterprise GCP-primary organisations.** Teams that have standardised on Google Cloud and use Vertex AI for model access can now run Codex CLI within their existing infrastructure boundary. Data stays in GCP; audit logs flow to Cloud Logging; IAM controls apply.

**Teams with data residency requirements.** GCP regions allow precise control over where inference requests are processed. For organisations with GDPR, HIPAA, or SOC 2 constraints on where code and context data can be processed, native Vertex AI support removes the need for a proxy tier.

**Multi-cloud engineering pods.** The observability sprint's multi-cloud credential work (AWS Bedrock default-on in v0.164.0, GCP Vertex AI built-in in v0.165.0) means a team can now run the same Codex workflow against different cloud backends in different environments without changing code.

The `gcp` provider joins `openai` and `bedrock` as the three built-in provider options. Switching between them requires only a `config.json` change; session state, preferences, and AGENTS.md configuration carry over unchanged.

---

### GPT-6.1 Terra in the Model Catalogue

v0.165.0 adds GPT-6.1 Terra to the Codex model catalogue alongside the existing GPT-6.1 Sol tier.

Terra sits above Sol in the GPT-6.1 family. The primary differentiation is instruction-following — Terra processes structured directives in AGENTS.md, system prompts, and tool descriptions more reliably than Sol. This is directly relevant to the AGENTS.md compliance regression reported in late October (GPT-6.1 Sol's pattern of acknowledging then re-violating explicit instructions). Terra addresses that regression at the model level.

Pricing and access:

- Available to Plus, Pro, Business, Enterprise, and Edu users in ChatGPT Work and Codex
- API: `gpt-6.1-terra` at \$1 per million input tokens, \$5 per million output tokens
- Half the price of GPT-6.1 Sol for API usage

Select Terra in Codex with:

```text
/model gpt-6.1-terra
```

Or set as default in config:

```json
{
  "model": "gpt-6.1-terra"
}
```

The `features.simplified_selection_hints` flag introduced alongside Terra in the v0.166.0 alpha sprint provides simplified model-selection hints in the TUI for users who want guided model choice rather than the full catalogue list.

---

### Session Picker with Repository Filtering

v0.165.0 overhauls the session picker for users working across multiple repositories.

The previous session picker presented a flat chronological list. For a developer working across five repositories in a given week, finding the right session required scrolling and recognising session titles rather than filtering by context.

The new session picker adds:

- **Repository filter** — type a repository name to narrow the list to sessions from that repo
- **Directory grouping** — sessions are grouped by repository, with the current repo's sessions at the top

This is a quality-of-life change with a measurable impact on multi-project workflows. In practice, the flat list became a liability as session counts grew — users running Codex daily across several projects would accumulate hundreds of sessions over months. Repository filtering makes those sessions navigable.

---

### Reasoning Effort Adjustable Mid-Session

v0.165.0 makes `--reasoning-effort` a live control rather than a startup parameter.

Previously, setting reasoning effort (`low`, `medium`, `high`) required either a command-line flag at launch or a configuration default. Changing effort during a session meant restarting. This created friction for sessions that naturally vary in complexity — a debugging pass followed by an architecture discussion followed by a routine code-generation task might warrant three different effort levels, but the cost of restarting was enough to keep most users on a single setting.

From v0.165.0, use the `/reasoning-effort` command mid-session:

```text
/reasoning-effort high
```

The change takes effect immediately on the next turn. Session state, conversation history, and all active configurations are preserved — only the reasoning budget changes.

**Practical pattern:** run `medium` as a default, switch to `high` when tackling a complex design decision or debugging a subtle concurrency issue, return to `medium` for routine generation. The cost difference between effort levels is significant enough that this control pays for itself quickly in high-volume agentic workflows.

---

### Memory v2 Default for Fresh Profiles

New Codex CLI installs and fresh configuration profiles now initialise with Memory v2 enabled by default.

Memory v2 is the structured retrieval layer introduced in earlier sprint cycles. Key differences from Memory v1:

| Feature | Memory v1 | Memory v2 |
|---------|-----------|-----------|
| Retention period | Session-scoped | Long-lived, cross-session |
| Retrieval method | Sequential scan | Structured, context-keyed |
| Memory size limit | ~8 KB | ~64 KB (configurable) |
| Codebase awareness | None | Project-aware by default |

For users already running Memory v2 (opt-in from earlier releases), nothing changes. For users who had not yet enabled it, new installs will receive Memory v2 behaviour automatically. Existing configurations that did not set a memory tier explicitly will continue with their current setting until manually updated.

To check your memory tier:

```text
/status
```

The memory configuration is shown in the status output. To explicitly set Memory v2 on an existing install:

```json
{
  "memory_tier": "v2"
}
```

---

### `codex doctor --network-only`

v0.165.0 adds a `--network-only` flag to the `codex doctor` diagnostic command.

The full `codex doctor` check covers filesystem permissions, sandbox configuration, credential validation, model connectivity, and MCP server reachability. In environments where filesystem and sandbox are known-good (e.g. a CI runner that was working yesterday), running the full diagnostic to chase a connectivity issue adds unnecessary noise.

```bash
codex doctor --network-only
```

The `--network-only` flag restricts the check to network-facing components: API reachability, credential token validity, proxy configuration, and MCP server connectivity. It runs faster and produces a focused output useful for network-layer debugging.

---

### Catalogue Headings in Saved Command-Prefix Notices

A UX detail worth noting: v0.165.0 includes catalogue headings in the notices shown when a saved command prefix is active. When a preset or saved command prefix is running, the TUI now displays the catalogue heading associated with that prefix, making it immediately clear which saved workflow is active without requiring the user to remember the prefix content.

---

## Sprint Summary

v0.165.0 was a four-day sprint with a clear internal theme: graduating the observability infrastructure from alpha to stable. The sprint that opened in v0.165.0-alpha.1 added turn-level tracing, streaming instrumentation, and tool-dispatch logging — making Codex's internal operation legible for the first time. That work is now stable.

The additions — GCP Vertex AI, Terra, Memory v2 default, mid-session reasoning effort — are not incidental. They are the commercial and operational layer that the observability infrastructure now supports: enterprises need multi-cloud support with audit trails (GCP Vertex AI), compliance-critical deployments need reliable instruction-following (Terra), long-running agentic workflows need persistent memory (Memory v2), and dynamic workloads need real-time cost control (reasoning effort mid-session).

v0.166.0-alpha.1 opened within three hours of v0.165.0 going stable. The new sprint introduces GPT-6.1 Terra safety recourse in the TUI, multi-process session index reliability, and VS Code scoping improvements. v0.165.0 alpha.4 and alpha.5 landed on 28 October with internal refactoring. Stable for v0.166.0 is expected within the week.

---

## Upgrade

Update to v0.165.0:

```bash
npm install -g @openai/codex@latest
```

Verify:

```bash
codex --version
# 0.165.0
```

---

## Related Articles

- [Inside the Black Box: Codex CLI v0.165.0's Observability Sprint](../2026-10-26-v0165-observability-sprint-tracing-streaming-tool-dispatch-guardian-v2/)
- [GCP Vertex AI Provider: How v0.165.0-alpha.1 Expands Codex Beyond OpenAI's Infrastructure](../2026-10-23-gcp-vertex-ai-provider-codex-cli-alpha11-1/)
- [Codex CLI v0.164.0 Stable: /presets, Image Previews, Enterprise MCP GA, and Bedrock Default-On](../2026-10-23-codex-v0164-stable-release-presets-image-previews-enterprise-mcp-bedrock/)
- [Codex CLI v0.166.0-alpha.1: GPT-6.1 Terra, Safety Recourse TUI, and Session Reliability](../2026-10-27-codex-v0166-alpha1-gpt61-terra-safety-recourse-session-reliability/)

---

*Sources: [GitHub Releases — openai/codex](https://github.com/openai/codex/releases/tag/rust-v0.165.0), [Codex Updates October 2026 — Releasebot](https://releasebot.io/updates/openai/codex), [9to5Mac — OpenAI upgrades ChatGPT and Codex with GPT-6.1 Terra](https://9to5mac.com/2026/10/26/openai-upgrades-chatgpt-and-codex-with-new-gpt-6-1-terra-model/)*
