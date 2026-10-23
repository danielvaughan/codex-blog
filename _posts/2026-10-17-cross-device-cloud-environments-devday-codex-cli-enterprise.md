---
title: "Cross-Device Cloud Environments: What DevDay's Biggest Codex Announcement Actually Means"
parent: "Articles"
date: 2026-10-17T09:00:00+00:00
last_modified_at: 2026-10-23T10:06:16+01:00
nav_order: 1187
tags: ["codex-cli", "codex-cloud", "cloud-environments", "devday-2026", "enterprise", "cross-device", "mobile", "configuration"]
copy_edited: true
---

# Cross-Device Cloud Environments: What DevDay's Biggest Codex Announcement Actually Means

![Sketchnote diagram for: Cross-Device Cloud Environments: What DevDay's Biggest Codex Announcement Actually Means](/sketchnotes/articles/2026-10-17-cross-device-cloud-environments-devday-codex-cli-enterprise.png)


---

> *The headline at DevDay 2026 was "close your laptop and pick it up from your phone". The mechanism that makes that possible — persistent cloud environments — is more architecturally significant than the demo suggested. This article explains what a Codex Cloud Environment actually is, how it differs from local worktrees, and what enterprise teams need to understand before adopting it.*

OpenAI's DevDay 2026 keynote on 29 September included a live demo: a developer handed off a running Codex task from a laptop to a mobile phone mid-execution. The session continued seamlessly — the agent's context, environment, and in-progress work all persisted across devices.[^1]

The mechanism enabling this handoff is the **Codex Cloud Environment**: a named, persistent execution environment that encapsulates your project's dependencies, secrets, configuration, and AGENTS.md in an OpenAI-managed container. Any device — or any Codex client — can connect to it. The laptop-to-phone demo is the consumer-friendly illustration of something with deeper implications for enterprise teams: a move from per-machine Codex configuration to centralised, team-shared environment governance.

This article explains what a Cloud Environment contains, how it differs from local sessions and `--worktree` execution, what the current v0.162.0-era limitations are, and how enterprise teams should think about the security model before publishing their first environment.


## What a Codex Cloud Environment Actually Contains

A Codex Cloud Environment is a snapshot of an execution context. When you publish an environment, you are capturing:

**Project environment**
The repository checkout (or a defined subset), installed dependencies, system packages, language runtimes, and build tooling. When a Codex task starts in a named environment, it inherits this state — no `npm install`, no `pip install`, no waiting for the setup script the local machine has already cached.

**Configuration and policy**
The AGENTS.md file(s) associated with the project, `config.toml` settings scoped to the environment, and approval policy configuration. Teams can enforce specific approval levels or Guardian configuration at the environment level rather than relying on per-developer local config.

**Secrets**
Secrets are stored in the environment's secrets vault, not in `.env` files. Three scopes are available: inline (session-only), personal (available to your identity across all your environments), and team shared (available to all team members who can access the environment). The vault is separate from the repo checkout and does not appear in version control or session transcripts.

**Named sessions**
Unlike local worktrees — which are ephemeral and tied to the machine that created them — Cloud Environment sessions are named and persistent. A session you started on your laptop this morning is still there when you open the mobile client this afternoon. The agent's context window, its in-progress work, and the checkpoint state are preserved in the cloud environment's session store.

What a Cloud Environment does **not** contain: local filesystem paths, `.env` files in the repo, or machine-specific credential stores. This is the intended design — it forces environment definitions to be explicit and portable rather than relying on local state that will not exist on another device.


## The Snapshot Model

The practical workflow for using Cloud Environments is:

1. Open the Codex Cloud UI (`ChatGPT → Work in → Cloud`)
2. Connect your repository and complete the chat-based onboarding (Codex researches the repo's requirements and proposes an environment definition)
3. Configure networking policy, secrets, and AGENTS.md overrides
4. **Publish** the environment as a named snapshot

Once published, every Codex task you start — CLI, IDE extension, or Codex app — that references this environment inherits the snapshot state. You are not setting up the environment per-task; you are booting a pre-configured container.

Flavio Copes documented this flow in his hands-on walkthrough using his "Waiting Lists" open-source project (published 1 October 2026).[^2] The key practical observation: the onboarding is chat-driven, not configuration-file-driven. Codex reads the repository and asks clarifying questions before proposing an environment definition — which lowers the barrier to setup but means the first-time experience is slower than experienced practitioners might expect.

**The bug to know about**: As of v0.162.0, there is a tracked issue where a published environment is not automatically applied to new Cloud tasks in some configurations. If you publish an environment and tasks still appear to be starting from a bare state, verify the environment is selected in the task creation dialogue. This is expected to be resolved in a coming stable release.[^3]


## Cross-Device Continuity: How It Works

The cross-device scenario works because both the session state and the execution environment live in OpenAI's infrastructure, not on the local machine. When you switch devices:

1. The new device connects to the Codex Cloud API
2. The API returns the list of named sessions associated with your account and the environment
3. You resume a named session — the agent's context is restored from the cloud session store
4. You can review what has been done, steer the agent, or hand off monitoring to someone else on the team

From the agent's perspective, a device switch is invisible. The task continues in the same environment with the same context. From the user's perspective, it is a reconnect — you are picking up an active session, not starting a new one.

This is architecturally different from a worktree handoff, where a human reviews the worktree's diff, commits what they want to keep, and starts a new session on another machine. Cloud Environment continuity is genuine session persistence: the model's context window is preserved across the handoff, not reconstructed from version-controlled state.

For the specific scenarios this enables — overnight runs monitored from mobile, handoffs between team members working in different time zones, executive visibility into long-running agent tasks — the cross-device model is more practical than any worktree-based alternative.


## Cloud Environments vs Local Worktrees

These are not competing features — they address different needs. The decision framework:

| Scenario | Cloud Environment | Local `--worktree` |
|---|---|---|
| Long-running tasks, laptop-closed | ✅ Native | ❌ Requires machine to stay running |
| Cross-device monitoring | ✅ Native | ❌ Not supported |
| Team-shared environment | ✅ Named environments + team vault | ❌ Per-machine config |
| Strict data-residency requirements | ❌ Data in OpenAI infra | ✅ Stays on-premise |
| Custom OS-level sandbox policies | ❌ Managed container | ✅ Full `sandboxPolicy` control |
| Offline work | ❌ Requires cloud connectivity | ✅ Works offline |
| Low-latency short tasks | ❌ Container boot overhead | ✅ Local startup is faster |
| Sensitive credential access | ✅ Secrets vault | ✅ Local `.env` or system keychain |

The hybrid workflow that combines both: use a local `--worktree` session for rapid exploration and short-cycle iteration; switch to a Cloud Environment for the overnight build, the slow test suite, or the task that should continue whether or not your laptop is open.


## Enterprise Considerations

For enterprise teams evaluating Codex Cloud Environments, three topics require deliberate assessment.

**Centralised environment = centralised attack surface**

In a per-machine model, each developer's Codex configuration is isolated. If one machine is compromised, the blast radius is limited to that developer's access. With a shared Cloud Environment, the secrets vault and AGENTS.md configuration are shared infrastructure. An attacker who gains access to the environment can read secrets, modify AGENTS.md policy, and observe all session activity.

The mitigations are the standard ones for shared infrastructure: environment-level access controls, audit logging of vault access, and treating the environment's AGENTS.md as a security-critical file subject to change review — not a convenience file that anyone on the team can edit.

**AGENTS.md as team policy**

In a Cloud Environment, the AGENTS.md file is the team's shared agent policy. It is the document that tells Codex what it is allowed to do, what it must ask about, and how to handle sensitive operations. In local sessions, each developer owns their own AGENTS.md. In a team Cloud Environment, AGENTS.md changes affect everyone.

This is an argument for version-controlling your Cloud Environment's AGENTS.md and treating changes to it with the same review process you would apply to a CI/CD configuration change. The `guardian_root_handoff_context` feature introduced in v0.160.0 is particularly relevant here: if Guardian is configured in the cloud environment, it will apply session-level instructions to all tasks in that environment.

**Token accounting and cost attribution**

Cloud Environment tasks run against your account's token budget. For teams with multiple developers running tasks concurrently in a shared environment, the token accounting model needs to be understood before billing surprises arrive. The `/usage` dashboard (introduced in v0.156.0) provides per-session and per-plugin attribution — establish a review cadence before running high-volume cloud tasks.


## Getting Started

For practitioners ready to create their first Cloud Environment:

1. Open `ChatGPT → Work in → Cloud` (requires ChatGPT Plus, Pro, or Team)
2. Connect your repository via the repo picker (GitHub integration required)
3. Complete the onboarding dialogue — Codex will inspect the repo and propose dependencies and configuration
4. Configure secrets: add any API keys or environment variables your tasks will need to the personal or team vault
5. Review the proposed AGENTS.md additions and networking policy
6. Publish the environment under a memorable name
7. Run your first cloud task and verify the environment snapshot is applied correctly (check the task startup log for "environment: [your-name]")

For teams: designate one person to own the initial environment definition, treat the first published snapshot as a draft, and iterate before sharing team vault access broadly.

Cloud Environments are the infrastructure layer that makes Codex useful as a persistent coding partner rather than a per-session tool. The cross-device demo was the attention-getter — the team-shared environment model is where the enterprise value actually lives.

---

[^1]: OpenAI DevDay 2026 keynote (29 September 2026, San Francisco). "Meet the all new Codex Cloud" — official product video. Available at: https://youtube.com/watch?v=7Bv68f5szSU

[^2]: Flavio Copes — "How to use Codex Cloud" (1 October 2026). Hands-on walkthrough of Cloud Environment setup using an open-source project. Available at: https://flaviocopes.com/codex-cloud/

[^3]: Codex CLI v0.162.0-alpha release tracking — "Published environment not applied to new Cloud tasks" bug. Available at: https://github.com/openai/codex/issues

[^4]: OpenAI Help Centre — "Using Codex Cloud" (September 2026). Available at: https://help.openai.com/en/articles/20001545-using-codex-cloud

[^5]: Codex CLI v0.160.0 release notes (1 October 2026) — Guardian review context, `guardian_root_handoff_context`, worktrees as default execution model. Available at: https://github.com/openai/codex/releases/tag/v0.160.0
