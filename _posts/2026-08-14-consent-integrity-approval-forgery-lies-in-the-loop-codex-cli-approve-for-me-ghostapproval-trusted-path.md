---
title: "Consent Integrity and the Lies-in-the-Loop Attack: Why Your Approval Dialog Is Not a Security Boundary — and What Codex CLI's --approve-for-me Actually Defends"
date: 2026-08-14T09:00:00+00:00
last_modified_at: 2026-09-01T12:09:24+01:00
tags: ["codex-cli", "security", "approval-policy", "consent-integrity", "WYSIWYS", "GhostApproval", "PreToolUse", "auto-review", "trusted-path", "LITL"]
---

# Consent Integrity and the Lies-in-the-Loop Attack: Why Your Approval Dialog Is Not a Security Boundary — and What Codex CLI's --approve-for-me Actually Defends


---

Every coding agent ships an approval dialog. You see a command, you press Enter, and the agent executes it. The implicit promise is that you approved *what runs*. In June 2026, Weng formalised the gap between that promise and reality: the approval dialog is narrated by the agent itself, and the narration is forgeable [^1]. A month later, Wiz's GhostApproval disclosure proved the attack works against six production agents [^2]. This article traces the research, maps it to Codex CLI's approval architecture in v0.147.0, and identifies where the defence still has holes.

## The Lies-in-the-Loop Attack

Weng's paper introduces the **Lies-in-the-Loop (LITL)** attack taxonomy. A compromised or prompt-injected agent can forge its own approval summary through four vectors [^1]:

- **Summary poisoning** — displaying a benign description for a malicious command
- **Padding/scroll-out** — inserting filler text so the dangerous payload scrolls beyond the visible approval region
- **Encoding/obfuscation** — wrapping payloads in base64, hex, or command substitution to defeat human inspection
- **TOCTOU swaps** — gaining approval for action A, then substituting action A' at execution time

The core insight is structural: when the agent both decides the action *and* narrates it to the human, the human is approving a description, not an operation. The approval dialog becomes a presentation layer over an opaque execution layer.

## Consent Integrity: WYSIWYS for Agents

The paper adapts the **What You See Is What You Sign (WYSIWYS)** property — a well-established concept in digital-signature law [^3] — to agent approval channels. Consent Integrity requires four conditions [^1]:

1. **Truthful rendering** — the human sees `R(A)`, computed by a trusted mediator from the actual action, not the agent's narration
2. **Faithfulness** — the rendering surfaces security-relevant facts: network egress, privilege escalation, credential reads, irreversible operations
3. **Trusted path** — approval delivery occurs over a channel the agent cannot intercept or overlay
4. **Bind-to-execution** — only the approved action executes; no post-approval substitutions

Since no static analyser can soundly classify every possible shell command, the paper settles for **analyser-relative Consent Integrity**: flag uninspectable actions rather than silently approving them.

## The Tradeoff Frontier

The prototype Consent-Integrity Mediator (CIM) — roughly 500 lines of dependency-free Python — was evaluated against two independent corpora [^1]:

| Corpus | Size | Flagged | Silent pass |
|--------|------|---------|-------------|
| GTFOBins (trusted-tool abuse) | 1,330 commands | 90.0% | 10.0% |
| tldr-pages (normal usage) | 28,798 commands | 95.9% | 4.1% |

The 87.0% uninspectable rate on tldr-pages reveals the central tension: shrinking the trust list drives silent passes towards zero but inflates approval prompts towards 100%, creating approval fatigue. Expanding it relieves fatigue but reintroduces silent passes. The authors are explicit: *"a boundary-only static mediator cannot escape the trade-off: it can choose where on the curve to sit... but it cannot occupy both ends at once"* [^1].

```mermaid
graph LR
    A["Smaller Trust List"] -->|"Silent passes → 0%"| B["Over-prompting<br/>≈100% flagged"]
    C["Larger Trust List"] -->|"Approval fatigue ↓"| D["Silent passes ↑<br/>Abuse surface widens"]
    B -.->|"Approval fatigue"| E["Users stop reading<br/>prompts"]
    D -.->|"GTFOBins abuse"| F["Trusted tools weaponised"]
    style E fill:#f96,stroke:#333
    style F fill:#f96,stroke:#333
```

## GhostApproval: The Theory Hits Production

Five weeks after Weng's paper, Wiz disclosed GhostApproval — a symlink-based attack combining CWE-61 (symlink following) with CWE-451 (UI misrepresentation) across six production agents [^2]:

| Agent | Behaviour | CVE |
|-------|-----------|-----|
| Amazon Q Developer | Pre-authorisation writes | CVE-2026-12958 |
| Claude Code | UI misrepresentation — agent knew the real target; user saw only the symlink name | — |
| Augment | Silent reads and writes without consent | — |
| Cursor | Backend symlink following after user accepts diff | CVE-2026-50549 |
| Google Antigravity | Displayed symlink path, not resolved path | Pending |
| Windsurf | Pre-authorisation RCE — writes before approval appears | — |

The Claude Code case is the most instructive for Codex CLI developers. The agent's internal chain-of-thought explicitly stated *"this is a symbolic link to the Claude settings file,"* yet the confirmation prompt asked simply: *"Make this edit to `project_settings.json`?"* [^2]. This is a textbook LITL summary-poisoning attack — not by a malicious adversary, but by the agent's own rendering pipeline failing to surface a security-relevant fact (the symlink target).

## Codex CLI's Approval Architecture in v0.147.0

Codex CLI's approval system operates through two orthogonal controls: `approval_policy` (when to stop and ask) and `sandbox_mode` (what generated commands can access) [^4]. The v0.147.0 release adds the `--approve-for-me` flag, which routes approval requests through an automatic-review subagent — the Guardian [^4] [^5].

```mermaid
flowchart TD
    A["Agent proposes action"] --> B{"approval_policy"}
    B -->|"on-failure-only"| C["Execute in sandbox"]
    B -->|"unless-allow-listed"| D{"Action in allow-list?"}
    B -->|"on-request"| E{"--approve-for-me?"}
    D -->|Yes| C
    D -->|No| F["Human approval prompt"]
    E -->|No| F
    E -->|Yes| G["Guardian auto-review"]
    G -->|"Low risk"| C
    G -->|"High risk"| F
    F --> H["Human sees rendered action"]
    H -->|Approve| C
    H -->|Reject| I["Action blocked"]
```

### Where Codex CLI Already Has Consent Integrity Properties

Several v0.147.0 features address the LITL attack surface:

- **Guardian auto-review** checks for data exfiltration, credential probing, persistent security weakening, and destructive actions before approving [^4]. This is a form of faithfulness rendering — surfacing security-relevant facts — albeit performed by an LLM subagent rather than a static analyser.
- **Secret redaction** strips secrets and bearer tokens from displayed commands and conversation history [^5], preventing credential leakage through the approval channel.
- **Hash binding at the session level** — tool approvals are centralised in the Session component, and turn environment permissions govern execution [^5], providing a form of bind-to-execution.
- **PreToolUse hooks** allow project-level static analysis before execution, enabling teams to implement their own consent-integrity checks [^4].
- **Project trust prompts** — Codex presents a one-time trust decision per repository, and untrusted projects skip all project-scoped `.codex/` layers including hooks, rules, and skills [^4]. This is a coarse-grained trusted-path mechanism.

### Where the Gaps Remain

Mapping the four Consent Integrity conditions to Codex CLI's current implementation reveals specific gaps:

**Truthful rendering** — Codex CLI's approval dialog shows the raw command, not a mediator-rendered version. If the command contains encoded payloads, symlinks, or indirection, the human sees the surface syntax, not the resolved operation. The Guardian subagent partially compensates, but it is itself an LLM — vulnerable to the same obfuscation techniques it tries to detect.

**Faithfulness** — there is no systematic enumeration of danger facts (network egress, privilege escalation, credential access) in the approval display. The Guardian checks for these patterns, but its classifications are probabilistic, not exhaustive.

**Trusted path** — the approval dialog renders in the same terminal session the agent controls. A sufficiently compromised agent could, in theory, manipulate terminal output. The v0.147.0 plugin isolation (denied network access on policy failure) [^5] reduces but does not eliminate this surface.

**Symlink resolution** — GhostApproval specifically exploited file-path display without symlink resolution [^2]. Codex CLI's sandbox mode restricts file-system access, which mitigates the attack vector, but the approval dialog itself does not resolve or display the actual target path of symlinks. **This is the most directly actionable gap.**

## Practical Hardening for Codex CLI Projects

Given the current state of the art, teams using Codex CLI in production should layer defences rather than relying on the approval dialog alone:

### 1. PreToolUse Hook for Symlink Detection

```bash
#!/usr/bin/env bash
# .codex/hooks/pre-tool-use-symlink-check.sh
# Reject file operations targeting symlinks outside the workspace

TARGET="$1"
if [ -L "$TARGET" ]; then
    RESOLVED=$(readlink -f "$TARGET")
    WORKSPACE=$(pwd)
    if [[ "$RESOLVED" != "$WORKSPACE"* ]]; then
        echo "BLOCKED: $TARGET is a symlink to $RESOLVED (outside workspace)" >&2
        exit 2  # exit 2 = feedback to agent
    fi
fi
```

### 2. Restrict Sandbox Mode

```toml
# .codex/config.toml
[sandbox]
sandbox_mode = "workspace-write"  # prevents writes outside project root
```

### 3. Use --approve-for-me with Named Profiles

```bash
# High-security profile for untrusted repositories
codex --profile strict --approve-for-me exec "review this PR"
```

The `--approve-for-me` flag routes through Guardian auto-review, which — despite being LLM-based rather than static — currently achieves 99.1% approval accuracy and 99.3% prompt-injection recall in OpenAI's benchmarks [^4].

### 4. PostToolUse Verification Hook

```bash
#!/usr/bin/env bash
# .codex/hooks/post-tool-use-verify.sh
# Verify no files outside workspace were modified

MODIFIED=$(git diff --name-only 2>/dev/null)
for f in $MODIFIED; do
    REAL=$(readlink -f "$f" 2>/dev/null || echo "$f")
    if [[ "$REAL" != "$(pwd)"* ]]; then
        echo "ALERT: File outside workspace modified: $REAL" >&2
        exit 2
    fi
done
```

## The Defence Treadmill

Weng's paper rejects the framing of consent integrity as a solved problem. Each improvement — adding danger-fact detection, implementing hash binding, resolving symlinks — moves the frontier incrementally. Implementing new boundary facts reduced GTFOBins silent passes from 15.0% to 10.0% [^1]. But richer obfuscation techniques, opaque binaries without provenance, and capabilities outside the analyser's enumeration remain blind spots.

For Codex CLI, the practical implication is that the Guardian auto-review subagent is a point on this curve, not the end of it. Its strength is breadth (LLM-based classification covers more patterns than a static analyser); its weakness is the same thing (LLM classification is probabilistic and gameable). The answer is not to choose between static analysis and LLM review but to layer both — PreToolUse hooks for known-bad patterns, Guardian auto-review for novel threats, and sandbox restrictions as the hard boundary beneath both.

The approval dialog was never a security boundary. The sooner we stop treating it as one, the sooner we can build the layered defences that actually work.

## Citations

[^1]: Weng, X. (2026). "What You Approve Is What Executes: Consent Integrity for Black-Box LLM Agents." arXiv:2606.02668. [https://arxiv.org/abs/2606.02668](https://arxiv.org/abs/2606.02668)

[^2]: Wiz Research. (2026, July 8). "GhostApproval: A Trust-Boundary Gap in AI Coding Assistants." Wiz Blog. [https://www.wiz.io/blog/ghostapproval-a-trust-boundary-gap-in-ai-coding-assistants](https://www.wiz.io/blog/ghostapproval-a-trust-boundary-gap-in-ai-coding-assistants)

[^3]: "What You See Is What You Sign (WYSIWYS)." Wikipedia. [https://en.wikipedia.org/wiki/WYSIWYS](https://en.wikipedia.org/wiki/WYSIWYS)

[^4]: OpenAI. (2026). "Codex CLI Guardian Approval: Configuring Auto-Review Policies." Codex CLI Documentation. [https://codex.danielvaughan.com/2026/04/20/codex-cli-guardian-approval-configuring-auto-review-policies/](https://codex.danielvaughan.com/2026/04/20/codex-cli-guardian-approval-configuring-auto-review-policies/)

[^5]: OpenAI. (2026, August 7). "Release 0.147.0." GitHub. [https://github.com/openai/codex/releases/tag/rust-v0.147.0](https://github.com/openai/codex/releases/tag/rust-v0.147.0)
