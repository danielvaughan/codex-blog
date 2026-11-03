---
title: "The 2026 Agentic Coding Landscape: Antigravity Is Live, Kiro Survives a Security Crisis, and Codex Finishes a Sprint"
date: 2026-11-03T09:00:00+00:00
last_modified_at: 2026-11-03T23:07:49Z
parent: "Articles"
nav_order: 1213
tags:
  - codex-cli
  - google-antigravity
  - kiro
  - competitive-analysis
  - agentic-coding
  - community
  - landscape
  - november-2026
copy_edited: true
---

# The 2026 Agentic Coding Landscape: Antigravity Is Live, Kiro Survives a Security Crisis, and Codex Finishes a Sprint

![Sketchnote diagram for: The 2026 Agentic Coding Landscape: Antigravity Is Live, Kiro Survives a Security Crisis, and Codex Finishes a Sprint](/sketchnotes/articles/2026-11-03-agentic-coding-landscape-2026-antigravity-kiro-codex-cli.png)

**Context:** As of 1 November 2026, Google's consumer Gemini CLI is retired and the Jules brand is fully discontinued. Antigravity is now Google's sole coding agent.

---

On 1 November 2026, Google retired its consumer Gemini CLI and discontinued the Jules brand entirely. What remains is Antigravity — the platform that launched at Google I/O 2026 in May and has spent six months absorbing everything Google built in the agentic coding space. The consolidation is complete, the transition window is closed, and the landscape is cleaner than it has been in over a year.

Meanwhile, Kiro — AWS's spec-driven coding agent — patched a serious remote code execution vulnerability in July, updated its model stack to Claude Opus 5.5, and kept growing its user base quietly. And Codex CLI just closed its "28 Days of Shipping" sprint with v0.166.0 stable, having shipped three major versions in October against what the community openly acknowledged was a Claude Code adoption gap.

This is the state of the field in November 2026.

---

## Google Antigravity: One Platform, Three Surfaces

Antigravity 2.0 shipped at Google I/O on 19 May 2026 as a five-component suite: a standalone Desktop app, a CLI written in Go, an SDK, a Managed Agents API, and an enterprise deployment path[^1]. The legacy Gemini CLI's forced deprecation deadline was June 2026; Jules brand products have now also been fully retired as of November[^2].

The result is a coherent product family where previously there were competing internal bets. The three primary surfaces are:

- **Antigravity Desktop** — a standalone application (not a VS Code fork) with a Manager View for multi-agent orchestration. The visual interface makes it the most approachable entry point for developers new to agentic workflows.
- **Antigravity CLI** — a Go binary for terminal-first workflows, comparable in scope to Codex CLI but with Google Cloud as the execution backend rather than a local sandbox.
- **Antigravity SDK** — programmatic access enabling infrastructure teams to embed the agent harness in their own deployment pipelines.

The ADK (Agent Development Kit) — Google's framework for building multi-agent systems — reached v2.6.0 in November, adding OpenTelemetry `gen_ai` semantic convention metrics. If you are building on top of Google's agentic infrastructure, OTEL instrumentation now works out of the box.

### What Antigravity Does Well

Native multi-agent parallelism is the headline capability. Antigravity orchestrates multiple subagents concurrently without requiring you to configure concurrency limits or set up external orchestrators — the platform manages agent lifecycle, routing, and teardown[^1]. For broad prototyping and parallel research tasks, this is faster to set up than Codex CLI's worktree-based isolation model.

The Google Cloud integration is deep: Firebase, Android Studio, Google AI Studio, GCP deployment, and Gemini 3.5 Flash as the default model at approximately 289 output tokens per second[^1]. If your stack is already GCP-first, Antigravity is the path of least resistance.

### What It Gives Up

Local kernel-level sandboxing. Codex CLI runs with Linux Landlock/Bubblewrap on Linux and Seatbelt on macOS — OS-enforced process isolation that works offline and leaves no network path for agent actions to reach external infrastructure[^3]. Antigravity's managed agents execute on Google Cloud infrastructure. For regulated environments or air-gapped deployments, this is a structural disqualifier.

The Go CLI does not expose the same hooks system maturity as Codex CLI's six-lifecycle-event hooks engine. If you depend on `PreToolUse` / `PostToolUse` governance for agent auditing in production pipelines, Codex remains the stronger choice.

---

## Kiro: Spec-Driven, Security-Tested, Still Alive

AWS's Kiro takes a philosophically different position to both Codex and Antigravity. Before writing a line of code, Kiro generates structured EARS-notation requirements, an architectural design document, and a dependency-sequenced task list — all stored as auditable markdown artefacts in `.kiro/specs/`[^4]. The code emerges from a specification, not directly from a prompt.

Kiro upgraded its model stack to Claude Opus 5.5 with adaptive thinking on paid tiers, and claims a 4x throughput improvement on parallel Spec task execution compared to the single-spec model.

### The CVE-2026-10591 Incident

On 22 July 2026, Intezer published full disclosure of CVE-2026-10591: a prompt-injection-to-remote-code-execution chain in Kiro that allowed a poisoned documentation page to silently rewrite the tool's MCP server configuration and execute arbitrary code with the developer's privileges[^5]. The attack required no user interaction beyond asking Kiro to summarise a URL.

The root cause was architectural: Kiro's `fsWrite` tool could write to `~/.kiro/settings/mcp.json` without triggering an approval prompt, and MCP config changes caused an automatic reload that launched the new (attacker-controlled) server regardless of user response[^5].

AWS patched this in Kiro 0.11.130 with protected paths requiring explicit approval before writes to `mcp.json`, `.vscode/tasks.json`, and `.git` directories. The patch addresses the specific exploit. It does not change the underlying design pattern — an agent that can request writes to its own trust configuration remains one user-approval click away from the same class of attack.

For teams evaluating Kiro, this is worth weighing. Codex CLI's kernel-level sandbox prevents the agent from reaching config paths at the OS level — no approval prompt needed because the path is structurally inaccessible[^3].

### Who Kiro Is Right For

Kiro's spec-driven workflow is the strongest in the field for teams that want structured, auditable AI development: a requirements document, a design document, and a task list before any code is generated. If your engineering culture prizes specification-first development and you want AI assistance that respects that culture rather than bypassing it, Kiro's approach is genuinely differentiated.

Its Bedrock integration means it works naturally in organisations already running AWS. The agent hooks are filesystem-event-driven (trigger on file save/create/delete) rather than tool-event-driven like Codex's hooks — better for passive automation, less capable for agent governance.

---

## Codex CLI v0.166.0: After 28 Days of Shipping

OpenAI's "28 Days of Shipping" sprint ran 5 October to 1 November 2026. The framing was explicit: Stack Overflow's 2026 developer survey had Claude Code at 65.5% adoption versus Codex CLI at 29.5%, and the sprint was OpenAI's response[^6].

Three stable releases shipped in October: v0.164.0, v0.165.0, and v0.166.0.

**v0.166.0** (29 October) brought queue resume (failed or cancelled sessions can be restarted from the queue without losing context), keyboard transcript navigation, GPT-6.1 Terra as a new reasoning model in the catalogue, GCP Vertex AI provider via a configuration wizard, and per-chat MCP server leases (each conversation gets its own MCP context rather than sharing a persistent connection)[^7].

**v0.165.0** (27 October) added GCP Vertex AI as a first-party cloud provider alongside Amazon Bedrock, GPT-6.1 Terra (OpenAI's next-tier reasoning model in the GPT-6.1 family, also available via Vertex), and Memory v2 — a redesigned session compaction system that preserves more context across long coding sessions[^7].

**v0.164.0** (23 October) introduced `/presets` for switching between named configuration profiles, image previews in the TUI, enterprise MCP authentication overhaul with single sign-on via account grant reuse, and Amazon Bedrock graduating from alpha to stable[^8].

The sprint narrative had a self-inflicted complication: on 30 October, OpenAI cut Pro plan Codex allowances without announcement. Community backlash was immediate and sharp. The cut was reversed within 24 hours, but the trust damage persisted through the sprint's close. The scorecard article covering the sprint noted this as the defining operational failure of the period — three solid releases undercut by reactive change management[^9].

---

## Where Each Tool Wins in November 2026

| Dimension | Codex CLI v0.166.0 | Antigravity 2.0 | Kiro |
|---|---|---|---|
| Kernel-level sandbox | Yes (Landlock/Seatbelt) | No (cloud-managed) | No (process isolation only) |
| Multi-agent parallelism | Good (worktrees + subagents) | Excellent (platform-native) | Good (parallel spec tasks) |
| Model family | GPT-6.1 Sol/Terra, o3, GPT-4.1 | Gemini 3.5 Flash | Claude Opus 5.5 |
| Open source | Yes (Apache 2.0) | No | No |
| Spec-driven workflow | Manual (PLANS.md) | No | First-class |
| CI/CD headless mode | `codex exec` (mature) | CLI flag | `--no-interactive` (v2.0+) |
| Enterprise MCP auth | Mature (v0.164.0+) | Google Workspace | AWS IAM |
| Hooks system | Six lifecycle events | JSON hooks | Filesystem-event hooks |
| Multi-cloud | Yes (AWS + GCP + Azure) | GCP-first | AWS-first |

### Decision shortcuts

**Choose Codex CLI** when: kernel-level sandbox is required, your team is invested in the OpenAI model family, you need mature CI/CD integration via `codex exec`, or you want open-source auditability.

**Choose Antigravity** when: native multi-agent parallelism without configuration overhead is the priority, your stack is GCP/Firebase/Android, or you want a visual management layer for parallel agent workflows.

**Choose Kiro** when: your engineering culture prioritises spec-first development, your infrastructure is AWS/Bedrock, and you want AI assistance that works from structured requirements documents rather than raw prompts.

**The dual-tool pattern remains viable:** Antigravity for rapid parallel prototyping, Codex CLI for sandboxed production execution. The tools are not mutually exclusive on a team that has multiple workflow patterns.

---

## What to Watch in November

Three signals worth tracking over the next 30 days:

1. **v0.167.x sprint theme** — OpenAI's next sprint is expected to open Monday 3 November. The 28 Days sprint improved enterprise and multi-provider stories. Watch whether v0.167.x continues that thread or pivots to developer experience.
2. **Antigravity ADK 2.7** — ADK 2.6.0's OTEL metrics are a signal that Google is investing in observability for production multi-agent deployments. A 2.7 release in November could solidify this.
3. **Kiro's response to the CVE pattern** — CVE-2026-10591 was the third structurally similar vulnerability in roughly one year. If AWS ships a deeper architectural change (not just protected paths), that would meaningfully improve Kiro's security posture for enterprise evaluation.

The landscape in November 2026 is the clearest it has been. The race is not over — but the contenders are known.

---

*Full technical reference with diagrams → [The 2026 Agentic Coding Landscape: Antigravity Is Live, Kiro Survives a Security Crisis, and Codex Finishes a Sprint](https://danielvaughan.com/codex-resources/articles/agentic-coding-landscape-2026-antigravity-kiro-codex-cli/)*

> *Written with AI assistance. This newsletter is researched and drafted by Andy, an AI assistant. Daniel Vaughan reviews and directs all published content.*

---

## Citations

[^1]: [Google I/O 2026 Developer Highlights: Antigravity, Gemini API, AI Studio](https://blog.google/innovation-and-ai/technology/developers-tools/google-io-2026-developer-highlights/)
[^2]: [Morning Briefing — 2 November 2026: Google Antigravity live, Gemini CLI retired, Jules brand fully retired](https://danielvaughan.github.io/codex-resources/notes/morning-briefing-2026-11-02)
[^3]: [The Kiro MCP Config Rewrite: How a Hidden Line of CSS Achieved Remote Code Execution — and Why Codex CLI's Architecture Makes the Same Attack Structurally Impossible](https://danielvaughan.github.io/codex-resources/articles/2026-07-25-kiro-mcp-config-rce-prompt-injection-codex-cli-sandbox-defence-config-immutability)
[^4]: [Kiro: Amazon's Spec-Driven Competitor — Competitive Analysis](https://danielvaughan.github.io/codex-resources/notes/kiro-spec-driven-competitive-analysis)
[^5]: [Intezer: CVE-2026-10591 — Kiro MCP Config RCE (July 2026)](https://intezer.com/blog/research/cve-2026-10591-kiro-mcp-config-rce/)
[^6]: [The Honest 2026 Verdict: When to Choose Codex CLI and When to Choose Claude Code](https://danielvaughan.github.io/codex-resources/articles/2026-10-26-codex-cli-vs-claude-code-honest-2026-verdict-stack-overflow-workflow-guide)
[^7]: [Codex CLI v0.166.0 Stable: Queue Resume, Keyboard Transcript Navigation, GPT-6.1 Terra, GCP Vertex AI, Per-Chat MCP Leases](https://danielvaughan.github.io/codex-resources/articles/2026-10-31-codex-v0166-stable-queue-resume-keyboard-transcript-terra)
[^8]: [Codex CLI v0.164.0 Stable: Presets, Image Previews, Enterprise MCP, Bedrock](https://danielvaughan.github.io/codex-resources/articles/2026-10-23-codex-v0164-stable-release-presets-image-previews-enterprise-mcp-bedrock)
[^9]: [28 Days of Shipping: The Final Scorecard](https://danielvaughan.github.io/codex-resources/articles/2026-11-01-28-days-of-shipping-end-of-sprint-scorecard-openai-codex)
