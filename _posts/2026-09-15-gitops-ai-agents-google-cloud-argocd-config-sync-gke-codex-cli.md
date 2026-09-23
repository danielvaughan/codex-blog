---
title: "GitOps for AI Agents on Google Cloud: ArgoCD, Config Sync and GKE"
parent: "Articles"
nav_order: 1159
date: 2026-09-15T08:00:00+00:00
last_modified_at: 2026-09-23T18:09:02+01:00
tags: ["codex-cli", "gitops", "gke", "argocd", "config-sync", "enterprise", "ci-cd", "agentic", "infrastructure", "google-cloud"]
---

# GitOps for AI Agents on Google Cloud: ArgoCD, Config Sync and GKE


---

When Google Cloud's developer advocate Richard Seroter added a GKE GitOps article to his daily reading list on 14 September 2026, the framing was instructive.[^1] The piece was not about Kubernetes configuration hygiene in the abstract — it was about applying GitOps discipline specifically to AI agent deployments: the harness containers, the sandboxed execution environments, the MCP server manifests, and the approval-mode configuration that governs what an agent can touch in production. For Codex CLI teams moving beyond a single developer's laptop and into a shared GKE cluster, this is precisely the problem.

GitOps is not a new pattern. The core idea — that a git repository is the single source of truth for cluster state, and that a controller continuously reconciles the cluster toward that state — has been operational in Kubernetes shops since at least 2019. What is new is the pressure that agentic workloads apply to GitOps assumptions. Agents are not stateless HTTP services. They hold session context, consume disproportionate CPU and memory during reasoning bursts, make outbound network calls to MCP servers and model APIs, and occasionally attempt to write to files they should not. Deploying them with the same manifests you use for a web backend invites surprises. This article maps the GitOps toolchain available on Google Cloud to the specific deployment characteristics of Codex CLI harnesses running on GKE.

## The Two GitOps Controllers on GKE

Google Cloud offers two production-grade GitOps controllers for GKE, and choosing between them is the first architectural decision.[^2]

**Config Sync** is Google's managed GitOps controller, available in standard GKE at no additional charge (Google dissolved the separate GKE Enterprise tier in September 2025; former Anthos capabilities including Config Sync are now included in base GKE). It synchronises cluster resources directly from a git repository — typically a Cloud Source Repository or GitHub — without requiring a separate operator deployment. Config Sync is tightly integrated with Google's policy enforcement stack: Policy Controller (OPA Gatekeeper) can block non-compliant manifests at sync time, before they ever reach the API server. For regulated enterprises running Codex CLI harnesses that must satisfy audit requirements, Config Sync's integration with Cloud Audit Logs and its native fleet management support are strong arguments in its favour.[^3]

**ArgoCD** is the CNCF-graduated open-source option, widely deployed across cloud providers and on-premises. On GKE it runs as a standard Kubernetes workload. ArgoCD's advantage over Config Sync is its application model: it groups related manifests into `Application` or `ApplicationSet` resources with explicit health checks, sync windows, and rollback capabilities. For Codex CLI teams managing multiple harness variants — a nightly bulk-processing harness, an interactive development harness, and a CI harness that runs on pull request events — ArgoCD's application-per-environment model provides clearer operational boundaries than Config Sync's flat namespace hierarchy.

Neither tool is universally superior. The practical choice for most Codex CLI teams on Google Cloud is: Config Sync if you are on GKE and want managed GitOps with minimal operational overhead; ArgoCD if you need fine-grained application lifecycle control or are deploying across multiple cloud providers.

## What Goes in the GitOps Repository

The design decision that most teams get wrong when first applying GitOps to agent workloads is scope: what should live in the GitOps repository, and what should live in the application repository?

For Codex CLI deployments on GKE, the GitOps repository — sometimes called the *config repo* or *ops repo* — should contain:

**Harness container manifests.** The `Deployment` or `Job` resources that define how Codex CLI harness containers run: resource requests and limits, service account bindings, environment variable references (pointing to Secret Manager references, not literal values), and the container image tag. The image tag is the deployment signal — a CI pipeline updates this tag in the config repo after a successful image build and push, and the GitOps controller propagates the change to the cluster.[^4]

**MCP server manifests.** If your harness connects to self-hosted MCP servers (filesystem proxies, internal API adapters, database connectors), their `Deployment` and `Service` manifests belong in the config repo alongside the harness manifests. This ensures that a harness version bump and an MCP server version bump can be coordinated in a single pull request and applied atomically, eliminating the class of failures where a new harness expects an MCP server capability that the deployed server version does not yet expose.

**NetworkPolicy resources.** Codex CLI harnesses should run with restrictive egress policies that enumerate the specific endpoints they are permitted to reach: the OpenAI API, designated MCP servers, and any internal services with documented approval in the threat model. A `NetworkPolicy` manifest in the config repo makes the agent's permitted network surface auditable and version-controlled. Any change to the egress allowlist goes through a pull request with a required reviewer.[^5]

**Approval mode ConfigMaps.** Codex CLI's approval mode — `--full-auto`, `--suggest`, `--auto-edit`, or the more granular per-tool approval policies available since v0.154.0 — should not be baked into the container image or passed as an untracked environment variable. A `ConfigMap` mounted at a known path, managed in the config repo, makes the approval posture of each deployed harness visible in git history and subject to the same review process as any other infrastructure change.

The application repository — where the Codex CLI harness code, `AGENTS.md`, skills, and hooks live — should not contain cluster manifests. The separation keeps the operational configuration reviewable by infrastructure engineers who may not be familiar with the harness codebase, and vice versa.

## Sync Windows and Agent Workload Patterns

GitOps controllers typically allow administrators to define *sync windows* — time ranges during which synchronisation is permitted or blocked. For stateless services, sync windows are a courtesy: you block deploys during peak traffic hours. For Codex CLI harnesses, sync windows are a correctness requirement.

Consider a nightly bulk-processing harness that runs from 22:00 to 06:00, consuming a long-running Codex CLI session to refactor a large codebase. If ArgoCD applies a manifest update at 02:00 that changes the harness container image, the running `Job` is not affected — Kubernetes `Job` pods are not updated in place — but any new pods spawned by a parallel harness instance will use the new image. The result is two concurrent harness versions operating on the same repository, potentially with conflicting assumptions about file structure or API shape.

The mitigation is a sync window that blocks all harness-related `Application` synchronisation between 21:00 and 07:00, enforced at the ArgoCD or Config Sync level rather than relying on individual teams to coordinate deploys. ArgoCD's sync window configuration is explicit and version-controlled in the `Application` manifest itself:

```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true
  automated:
    prune: true
    selfHeal: true
  syncWindows:
    - kind: deny
      schedule: "0 21 * * *"
      duration: 10h
      applications:
        - codex-nightly-harness
```

Config Sync does not have a native sync window feature as of September 2026; teams using Config Sync for this pattern typically gate the config repo pull request merge with a GitHub Actions workflow that enforces time-of-day constraints before the merge is permitted.[^6]

## Secrets and Model API Keys

The single most common security mistake in initial Codex CLI GKE deployments is placing the `OPENAI_API_KEY` — and any MCP server credentials — in a `ConfigMap` or as a plain environment variable in the `Deployment` manifest. Plain-text secrets in the config repo are a critical security finding that will surface in any SOC 2 or ISO 27001 audit.

The standard pattern on Google Cloud is External Secrets Operator (ESO) with Google Secret Manager as the backend.[^7] ESO watches for `ExternalSecret` resources in the cluster and materialises them as Kubernetes `Secret` objects, pulling the actual values from Secret Manager at sync time. The config repo contains only the `ExternalSecret` manifest — which references the secret by name, not by value — and the `Deployment` manifest references the `Secret` by name. The actual key material lives in Secret Manager, never in git.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: codex-api-credentials
  namespace: codex-harness
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcp-secret-store
    kind: ClusterSecretStore
  target:
    name: codex-api-credentials
    creationPolicy: Owner
  data:
    - secretKey: OPENAI_API_KEY
      remoteRef:
        key: codex-openai-api-key
        version: latest
```

Secret Manager's audit log records every access, providing the evidence trail that security auditors need to verify that key material was not accessed outside of authorised agent runs.

## Progressive Delivery with ArgoCD Rollouts

Codex CLI harness deployments are not zero-risk. A new version of a harness might introduce a regression in hook logic, a change to `AGENTS.md` that alters agent behaviour in unexpected ways, or a dependency bump that changes how MCP tools are resolved. Rolling out a new harness version to all environments simultaneously compounds the blast radius of any such regression.

ArgoCD integrates with Argo Rollouts to provide progressive delivery strategies — canary, blue-green, and analysis-based promotion — for Kubernetes workloads.[^8] For interactive Codex CLI harnesses running in a development cluster, a blue-green strategy is appropriate: the new harness version runs in parallel with the existing version; engineers opt in to the new version by updating a routing label on their namespace; the old version is retained for a configurable period before being pruned.

For CI harnesses that run on every pull request, a canary strategy based on a synthetic workload — a known task with a verified expected output — is more appropriate. ArgoCD Rollout's `AnalysisTemplate` can invoke a Kubernetes `Job` that runs the synthetic task against the new harness version and compares the output to a golden file stored in the config repo. If the analysis passes, the rollout proceeds automatically; if it fails, the rollout is aborted and the previous version is retained.

This pattern — canary promotion gated on synthetic task correctness — is the production equivalent of the test-harness-first discipline described in the book's Chapter 20 discussion of CI/CD for agentic workflows. The difference is that in a GitOps setting, the promotion gate is enforced by the deployment infrastructure rather than by developer discipline.

## Observability as a GitOps Concern

One aspect of agent deployments that the standard Kubernetes GitOps literature underweights is observability configuration. For Codex CLI harnesses, the observability stack is not optional instrumentation — it is the primary mechanism for detecting when an agent is behaving unexpectedly, consuming disproportionate resources, or making unapproved network calls.

On GKE, the relevant manifests include:

- `PodMonitor` resources (Prometheus Operator) that scrape harness-emitted metrics
- `ServiceMonitor` resources for MCP server metrics
- Cloud Logging `LogExclusion` resources that filter agent output logs before they reach long-term storage (harness outputs can be voluminous)
- OpenTelemetry `Collector` configuration (as a `ConfigMap`) defining the traces pipeline from harness containers to Cloud Trace

All of these belong in the config repo. Placing observability configuration in the GitOps repository means that a change to what is monitored, or at what retention policy, goes through the same pull request review as any other infrastructure change. It also means that when an agent-related incident is reviewed in a post-mortem, the state of the observability configuration at the time of the incident is recoverable from git history.

## The Config Repo as Compliance Evidence

For enterprises deploying Codex CLI in regulated environments — financial services, healthcare, public sector — the config repo is not just an operational convenience. It is compliance evidence.

A well-maintained config repo provides:

- **Change history**: Every modification to an agent's approval mode, network policy, or resource limits is timestamped, attributed to an author, and associated with a pull request. The PR includes the reviewer's approval, the CI checks that passed, and the deployment outcome.
- **Drift detection**: Both ArgoCD and Config Sync detect when the live cluster state has diverged from the git state — for example, if an operator applied a change directly via `kubectl` outside the GitOps workflow. Drift events can be surfaced as alerts and included in audit reports.
- **Rollback evidence**: When a harness version is rolled back following an incident, the rollback is itself a git commit, reviewable and auditable.

For enterprises pursuing ISO/IEC 42001 AI management system certification — an emerging requirement for enterprise AI deployments as of 2026 — the config repo's change history directly addresses several Annex A control requirements around AI system change management and operational monitoring.[^9]

---

GitOps is not a silver bullet for the operational challenges of Codex CLI on GKE. It does not solve the harder problems of session state management, context-window economics, or approval-mode policy design. But it does solve the tractable problem of making agent deployments auditable, reversible, and consistently reproducible — which is the prerequisite for everything else. On Google Cloud, the combination of Config Sync for fleet-wide policy enforcement and ArgoCD for application-level lifecycle control gives Codex CLI teams a mature, production-validated foundation that the ecosystem has been building and testing for several years before AI agents arrived to use it.

---

## Footnotes

[^1]: Seroter, R. (2026, September 14). *Daily Reading List #866*. richard.seroter.com. Retrieved 2026-09-15.
[^2]: Google Cloud. (2026). *GitOps on GKE: Config Sync and ArgoCD overview*. cloud.google.com/kubernetes-engine/docs/concepts/gitops. Retrieved 2026-09-15.
[^3]: Google Cloud. (2026). *Config Sync overview*. cloud.google.com/kubernetes-engine/config-sync/docs/overview. Retrieved 2026-09-15.
[^4]: Weaver, W. (2025). *GitOps: Versioned Infrastructure and Continuous Deployment*. O'Reilly Media.
[^5]: CNCF. (2026). *Kubernetes NetworkPolicy documentation*. kubernetes.io/docs/concepts/services-networking/network-policies. Retrieved 2026-09-15.
[^6]: GitHub. (2026). *ArgoCD sync windows*. argo-cd.readthedocs.io/en/stable/user-guide/sync_windows. Retrieved 2026-09-15.
[^7]: External Secrets Operator. (2026). *Google Secret Manager provider*. external-secrets.io/latest/provider/google-secrets-manager. Retrieved 2026-09-15.
[^8]: Argo Project. (2026). *Argo Rollouts — Progressive Delivery for Kubernetes*. argoproj.github.io/argo-rollouts. Retrieved 2026-09-15.
[^9]: ISO/IEC. (2023). *ISO/IEC 42001:2023 — Information technology — Artificial intelligence — Management system*. iso.org/standard/81230.html.
