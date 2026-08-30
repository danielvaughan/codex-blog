---
title: "Comment and Control: How a Single GitHub Issue Broke Three AI Agent CI/CD Pipelines — and What Codex CLI's Proxy Architecture Gets Right"
date: 2026-07-20T09:00:00+00:00
last_modified_at: 2026-08-30T20:10:28+01:00
tags: ["codex-cli", "security", "ci-cd", "prompt-injection", "github-actions", "supply-chain", "claude-code", "codex-action"]
---

# Comment and Control: How a Single GitHub Issue Broke Three AI Agent CI/CD Pipelines — and What Codex CLI's Proxy Architecture Gets Right


---

A single pull request title, written by an outside contributor with zero special access, simultaneously hijacked Anthropic's Claude Code Security Review agent, Google's Gemini CLI Action, and GitHub's Copilot Coding Agent. In each case, the agent exfiltrated the repository's secrets — API keys, GitHub tokens, cloud credentials — back through GitHub itself [^1]. The attack class, dubbed **Comment and Control**, represents the most consequential supply-chain threat to emerge from the agentic CI/CD era: prompt injection via GitHub metadata [^2].

This article traces the anatomy of the attack, examines why every major AI coding agent except one fell to the same structural flaw, and dissects how Codex CLI's `codex-action` GitHub Action takes a fundamentally different architectural approach to the problem.

## The Structural Flaw: Trusted Instructions Meet Untrusted Data

Every AI agent running inside GitHub Actions faces the same architectural tension: the workflow's system prompt tells the agent what to do, but the agent must also read pull request titles, issue bodies, and review comments to do its job. Current implementations do not distinguish between content the model should treat as instructions and content the model should treat as data [^2].

This is not a novel insight — prompt injection has been a known attack vector since 2022. What changed in 2026 is that organisations began wiring AI agents into CI/CD pipelines with write access to repositories, network access to external services, and the ability to execute arbitrary shell commands. The blast radius of a successful injection went from "the model says something wrong" to "the attacker owns your supply chain."

## Anatomy of the Claude Code GitHub Action Attack

In January 2026, security researcher RyotaK of GMO Flatt Security reported a critical vulnerability chain in Anthropic's `claude-code-action` [^3]. The attack exploited three independently fixable flaws that, when chained, produced full repository compromise.

### Flaw 1: The Bot Suffix Bypass

The `checkWritePermissions` function — the gate that decided whether a workflow trigger came from a trusted actor — contained a single devastating line:

```javascript
if (actor.endsWith("[bot]")) { return true; }
```

Any GitHub App, regardless of actual permissions, passed the check simply because its actor name ended in `[bot]`. An attacker could create a GitHub App, install it on their own repository, and use its installation token to create issues on any public repository running the Claude Code Action [^3].

### Flaw 2: Wildcard Contributor Access

Anthropic's own example workflows — the templates most adopters copied — configured `allowed_non_write_users: "*"`, permitting any external contributor to trigger agent workflows with full access to repository secrets [^3].

### Flaw 3: Environment Variable Exposure

Once the agent processed a prompt-injected issue body, the attacker's payload could read `/proc/self/environ` to harvest `ACTIONS_ID_TOKEN_REQUEST_TOKEN` and `ACTIONS_ID_TOKEN_REQUEST_URL`. These OIDC credentials enabled token exchange for Claude's GitHub App installation token — granting write access to code, issues, pull requests, and workflows [^3].

### The Supply Chain Cascade

The most severe consequence was recursive: `anthropics/claude-code-action` itself used a vulnerable agent-mode workflow. A successful exploit against Anthropic's own repository would inject malicious code into the action's source, propagating to every downstream repository depending on it [^3]. The vulnerability received a CVSS v4.0 score of 7.8 and a \$4,800 bounty [^3].

```mermaid
flowchart TD
    A["Attacker creates<br/>GitHub App"] --> B["App creates issue on<br/>target public repo"]
    B --> C["checkWritePermissions<br/>trusts [bot] suffix"]
    C --> D["Claude Code processes<br/>injected issue body"]
    D --> E["Reads /proc/self/environ<br/>harvests OIDC tokens"]
    E --> F["Exchanges OIDC for<br/>GitHub App install token"]
    F --> G["Write access to<br/>target repository"]
    G --> H["If target is claude-code-action:<br/>supply chain cascade"]

    style A fill:#ff6b6b,color:#fff
    style H fill:#ff0000,color:#fff
    style C fill:#ffa726,color:#fff
```

## The In-the-Wild Exploitation

The theoretical became concrete in February 2026 when a prompt-injected issue title against Cline's `claude-code-action` triage workflow let attackers steal an npm publish token and push an unauthorised `cline@2.3.0` package [^3]. This was not a proof of concept — it was a successful supply-chain compromise of a tool used by thousands of developers.

## Why Three Agents Fell Simultaneously

The Comment and Control research by VentureBeat demonstrated that a single malicious PR title could simultaneously compromise Claude Code, Gemini CLI Action, and Copilot Coding Agent [^1]. The failure was structural, not implementation-specific:

1. **All three agents processed untrusted GitHub metadata as model input** without sanitisation or privilege separation
2. **All three held secrets in environment variables** accessible to the agent's execution context
3. **All three granted the agent shell access** sufficient to exfiltrate those secrets

Microsoft's Threat Intelligence team published a detailed analysis confirming that the prompt injection pathway "allowed access to workflow secrets under specific conditions" and that "an adversary who previously needed write access to a trusted repository now needs only the ability to create an issue" [^4].

## The Agents Rule of Two

The Cloud Security Alliance formalised the defensive principle as the **Agents Rule of Two**: an AI-powered workflow should never hold all three of the following capabilities simultaneously [^2]:

1. Processing untrusted input (issue bodies, PR titles, comments)
2. Holding sensitive credentials (API keys, OIDC tokens, GitHub tokens)
3. Executing state-changing tools (Bash, file writes, network requests)

Any two of the three is defensible. All three together creates the Comment and Control attack surface.

## How Codex CLI's codex-action Differs Architecturally

OpenAI's `codex-action@v1` takes a structurally different approach to CI/CD agent security [^5]. The differences are not cosmetic — they reflect a fundamentally different trust architecture.

### The Responses API Proxy

The headline architectural distinction is credential isolation. When you provide an `openai-api-key` input, the action starts a **Responses API proxy** — a separate process that holds the API key and forwards requests on behalf of the CLI. The runner process never holds a raw `Authorization` header in its environment [^5].

This matters because the `/proc/self/environ` exfiltration technique that broke Claude Code's action requires the secret to exist in the agent's process environment. With Codex's proxy architecture, the agent's process simply does not contain the credential.

### Irreversible Privilege Dropping

The `safety-strategy` parameter defaults to `drop-sudo`, which "removes `sudo` before running Codex. This is irreversible for the job and protects secrets in memory" [^5]. The `unprivileged-user` option goes further, executing Codex under a dedicated low-privilege account.

Compare this with Claude Code's action, which ran the agent under the same user context that held all workflow secrets — the context that `/proc/self/environ` exposes.

### Actor Gating

Codex's action provides `allow-users` and `allow-bots` parameters for restricting which actors can trigger the workflow [^5]. This is an explicit allowlist model, contrasting with Claude Code's original `allowed_non_write_users: "*"` wildcard that permitted any contributor.

### Sandbox Modes

Three sandbox modes — `workspace-write`, `read-only`, and `danger-full-access` — constrain filesystem and network access within the agent itself [^5]. The documentation explicitly warns: "Don't rely on `read-only` alone to protect secrets" — an acknowledgement that defence in depth requires multiple layers.

```mermaid
flowchart LR
    subgraph "Claude Code Action (pre-fix)"
        A1["Agent Process"] --- B1["API Key in env"]
        A1 --- C1["GitHub Token in env"]
        A1 --- D1["Shell access"]
        A1 --- E1["Untrusted input"]
    end

    subgraph "Codex Action"
        A2["Agent Process<br/>(no credentials)"] --- D2["Sandboxed shell"]
        A2 --- E2["Untrusted input"]
        F2["Proxy Process<br/>(holds API key)"] -.->|"forwards requests"| A2
        G2["allow-users<br/>allowlist"] -.->|"gates triggers"| E2
    end

    style B1 fill:#ff6b6b,color:#fff
    style C1 fill:#ff6b6b,color:#fff
    style F2 fill:#4caf50,color:#fff
    style G2 fill:#4caf50,color:#fff
```

## What Codex CLI's Architecture Does Not Solve

Architectural honesty requires noting the gaps. The proxy isolates the OpenAI API key, but the `GITHUB_TOKEN` — the credential GitHub injects into every Actions runner — remains in the agent's environment. A successful prompt injection could still exfiltrate the workflow's GitHub token unless the `permissions` key in the workflow YAML restricts its scope [^5].

The sandbox modes constrain the agent's filesystem and network access, but prompt injection occurs at the model layer — before any sandbox or hook can intervene. If the model is persuaded to use its legitimate capabilities maliciously, sandboxing limits the blast radius but does not prevent the compromise.

⚠️ No current AI coding agent fully solves the instruction-data confusion problem at the model layer. All defences are architectural mitigations, not complete solutions.

## Hardening Your CI/CD Agent Workflows Today

Whether you use `codex-action`, `claude-code-action` (post-fix), or any other AI agent in CI/CD, the following practices reduce your attack surface:

### 1. Scope the GITHUB_TOKEN

```yaml
permissions:
  contents: read
  pull-requests: read
  issues: read
```

Never grant `write` permissions unless the workflow genuinely needs them. The `permissions` key restricts `GITHUB_TOKEN` scope at the workflow level [^6].

### 2. Gate External Contributions

```yaml
on:
  pull_request_target:
    types: [opened]
jobs:
  review:
    if: github.event.pull_request.author_association == 'MEMBER'
```

First-time-contributor gates prevent external pull requests from triggering agent workflows. Codex's `allow-users` parameter provides equivalent gating [^5].

### 3. Migrate to Short-Lived OIDC Tokens

Replace stored secrets with OIDC token federation. GitHub Actions, GitLab CI, and CircleCI all support OIDC federation with token lifetimes measured in minutes, not hours [^2].

### 4. Strip Bash from Review Agents

If your agent's job is code review, it does not need shell access. Set repository access to read-only and gate write access behind a human approval step [^4].

### 5. Add Environment Scrubbing

Post-fix `claude-code-action` (v1.0.94+) now scrubs environment variables in child processes and wraps the `gh` CLI to validate arguments against exfiltration patterns [^3]. If your agent framework does not provide equivalent scrubbing, implement it in your workflow.

## The Broader Pattern

Comment and Control is not the last attack class that will target AI agents in CI/CD. It is the first to demonstrate that **the CI/CD pipeline is now the primary supply-chain attack surface** for AI-assisted development. The attack requires no malware, no vulnerability in the agent's dependencies, no compromised credentials — only a well-crafted issue title.

The defensive lesson is architectural, not procedural: separate credentials from computation, constrain tool access to the minimum necessary, and never process untrusted input in a context that holds sensitive state. Codex CLI's proxy-based credential isolation and irreversible privilege dropping implement two of these three principles by default. The third — untrusted input isolation — remains an unsolved problem across the industry.

## Citations

[^1]: VentureBeat, "Three AI coding agents leaked secrets through a single prompt injection. One vendor's system card predicted it," July 2026. [https://venturebeat.com/security/ai-agent-runtime-security-system-card-audit-comment-and-control-2026](https://venturebeat.com/security/ai-agent-runtime-security-system-card-audit-comment-and-control-2026)

[^2]: Cloud Security Alliance, "AI Agent Prompt Injection: The New CI/CD Supply Chain Threat," Lab Space Research Note, 2026. [https://labs.cloudsecurityalliance.org/research/csa-research-note-claude-code-github-action-prompt-injection/](https://labs.cloudsecurityalliance.org/research/csa-research-note-claude-code-github-action-prompt-injection/)

[^3]: RyotaK / GMO Flatt Security, "Poisoning Claude Code: One GitHub Issue to Break the Supply Chain," June 2026. [https://flatt.tech/research/posts/poisoning-claude-code-one-github-issue-to-break-the-supply-chain/](https://flatt.tech/research/posts/poisoning-claude-code-one-github-issue-to-break-the-supply-chain/)

[^4]: Microsoft Security Blog, "Securing CI/CD in an agentic world: Claude Code Github action case," June 2026. [https://www.microsoft.com/en-us/security/blog/2026/06/05/securing-ci-cd-in-agentic-world-claude-code-github-action-case/](https://www.microsoft.com/en-us/security/blog/2026/06/05/securing-ci-cd-in-agentic-world-claude-code-github-action-case/)

[^5]: OpenAI, "Codex GitHub Action," ChatGPT Learn documentation, 2026. [https://developers.openai.com/codex/github-action](https://developers.openai.com/codex/github-action)

[^6]: GitHub, "Permissions for the GITHUB_TOKEN," GitHub Actions documentation, 2026. [https://docs.github.com/en/actions/security-for-github-actions/security-guides/automatic-token-authentication](https://docs.github.com/en/actions/security-for-github-actions/security-guides/automatic-token-authentication)
