---
title: "Built-In Google Cloud Vertex AI: Codex CLI Adds a Native GCP Provider"
date: 2026-10-23T09:00:00+00:00
last_modified_at: 2026-10-24T11:34:40+01:00
parent: "Articles"
nav_order: 1201
tags:
  - codex-cli
  - v0.164.0
  - google-cloud
  - vertex-ai
  - gcp
  - model-providers
  - enterprise
  - configuration
copy_edited: true
---

# Built-In Google Cloud Vertex AI: Codex CLI Adds a Native GCP Provider

![Sketchnote diagram for: Built-In Google Cloud Vertex AI: Codex CLI Adds a Native GCP Provider](/sketchnotes/articles/2026-10-23-gcp-vertex-ai-provider-codex-cli-alpha11-1.png)

**Release track:** v0.164.0-alpha.11.1 (23 October 2026, 01:58 UTC)
**Key PR:** #56265
**Impact:** Google Cloud users, enterprise GCP deployments, multi-cloud Codex workflows

---

A single PR landed quietly in a hotfix branch on 23 October, shortly before the v0.164.0 stable release: Codex CLI now ships a **built-in Google Cloud Vertex AI provider**. Set `model_provider = "gcp"` in your configuration and Codex will route inference through Vertex AI without any custom HTTP adapter or third-party wrapper.

The feature did not make the v0.164.0 stable cut — the hotfix was branched off alpha.11 after the alpha.12 main branch had already frozen for stable — but it is merged and tested, and is expected to ship in v0.165.0.

---

## How it works

### Configuration

Add a single line to your Codex config to switch the active provider:

```toml
model_provider = "gcp"
```

Codex derives the Vertex AI endpoint from two environment variables:

| Variable | Purpose | Default |
|---|---|---|
| `GOOGLE_CLOUD_PROJECT` | GCP project ID | required |
| `GOOGLE_CLOUD_LOCATION` | Vertex AI region | `"global"` |

The `global` location default means you can get started with only a project ID set.

### Authentication

Credentials are obtained by shelling out to the `gcloud` CLI:

```bash
gcloud auth application-default print-access-token
```

This follows Application Default Credentials conventions familiar to any GCP practitioner. If you need finer control, the provider supports three overrides under `model_providers.gcp`:

- `base_url` — override the derived Vertex endpoint entirely
- `auth` — supply a custom auth mechanism
- `http_headers` — inject additional request headers

### Model catalogue

Codex ships a **static model catalogue tailored to Vertex AI** when the GCP provider is active. The catalogue filters out models that Vertex does not expose and surfaces the Vertex-specific model identifiers you would use in the Google Cloud console.

Standard Codex features that work with the GCP provider:

- Function tools
- Tool namespacing
- Client-side tool search
- Remote compaction V2

Features explicitly disabled (Vertex AI does not support them):

- Reasoning summaries
- Responses Lite
- Custom tools (server-side)
- Web search
- Image generation

### Account reporting

The app-server protocol gains a new `"gcp"` account type. This means the Codex UI can display your GCP project context alongside OpenAI and AWS Bedrock accounts — account reads do not require OpenAI authentication and credentials are not resolved at account-read time.

---

## Why this matters

Codex CLI has been OpenAI-first by design, with Bedrock arriving as the first major alternative provider. The GCP Vertex AI provider completes the major cloud triad:

| Provider | Config key | Auth mechanism |
|---|---|---|
| OpenAI | `openai` (default) | API key / OAuth |
| Amazon Bedrock | `bedrock` | AWS credentials / wizard |
| Google Cloud Vertex AI | `gcp` | Application Default Credentials |

For organisations already invested in Google Cloud — Vertex AI model deployments, Cloud Run workloads, BigQuery pipelines — this removes the last friction point for adopting Codex CLI as an agentic layer. No custom provider configuration, no HTTP shim: just a project ID and a `gcloud` login.

---

## What to expect next

Because this PR landed on an alpha.11 hotfix branch rather than the main development line, it will need to be merged or cherry-picked into the v0.165.0 alpha sprint before shipping in a stable release. Watch for `model_provider = "gcp"` to appear in the v0.165.0 release notes.

If you want to try it now, build from the `alpha.11.1` tag or wait for v0.165.0-alpha.1.

---

*Source: [PR #56265](https://github.com/openai/codex/pull/56265) — merged into v0.164.0-alpha.11.1 (23 October 2026, 01:58 UTC).*
