---
title: "Squad: Repository-Native Multi-Agent Orchestration and How It Compares to Codex CLI"
date: 2026-04-11T06:30:00+01:00
tags:
  - squad
  - multi-agent
  - github-copilot
  - orchestration
  - comparison
  - architecture
  - brady-gaster
---

# Squad: Repository-Native Multi-Agent Orchestration and How It Compares to Codex CLI

Brady Gaster's **Squad** is an open-source, TypeScript-based framework that provides multi-agent AI orchestration on top of GitHub Copilot[^1]. Where Codex CLI gives you a single powerful agent in a terminal, Squad gives you an entire *team* of specialised agents that live in your repository as plain-text files. The two tools solve different problems — and understanding the difference illuminates where AI-assisted development is heading.

## What Squad Is

Squad scaffolds a `.squad/` directory into your repository containing agent charters, routing rules, shared decision logs, and persistent memory files[^1][^2]. When you describe a task, a coordinator agent analyses the work, routes it to specialist agents (frontend, backend, tester, scribe), and those agents execute in **parallel** — each with its own context window, charter, and accumulated project history.

The key architectural idea: instead of one context window doing everything, Squad **replicates** repository context across multiple specialists. With Copilot's token budget distributed across five agents, the coordinator can leverage approximately one million tokens of collective reasoning capacity[^2].

Everything is committed to Git. Anyone who clones the repo gets the full team with all accumulated knowledge. There is no external service, no cloud dashboard, no vendor lock-in beyond GitHub Copilot itself.

### The `.squad/` Directory

```
.squad/
  team.md              # Roster — who's on the team
  routing.md           # Who handles what
  decisions.md         # Shared brain — architectural decisions
  ceremonies.md        # Sprint ceremonies config
  agents/
    lead/
      charter.md       # Identity, expertise, voice
      history.md       # What they know about YOUR project
    frontend/
      charter.md
      history.md
    backend/
      charter.md
      history.md
    tester/
      charter.md
      history.md
    scribe/
      charter.md       # Silent memory manager
  skills/              # Compressed learnings
  log/                 # Session history (searchable archive)
```

### How a Task Flows

1. You describe what you are building ("Build the login page")
2. The coordinator loads repository context and routes work
3. Multiple specialist agents spawn **simultaneously** in parallel — not sequentially
4. Each agent works in its own context window with its own charter and history
5. The Scribe silently logs everything to `decisions.md` and session logs
6. The Tester can **reject work**, forcing a *different* agent to provide a fresh perspective — not the original author revising its own output[^2]
7. The coordinator chains follow-up work
8. You review the integrated result

This review enforcement — where the Tester rejects work and a fresh agent provides the fix — is one of Squad's most interesting design choices. It prevents the blind spots that occur when a single agent reviews its own output.

## What Codex CLI Is

Codex CLI is OpenAI's open-source terminal-based coding agent[^3]. It runs as a single agent process in your terminal, authenticates via a ChatGPT subscription or API key, and executes tasks by reading files, writing code, and running shell commands inside a sandboxed environment.

Codex CLI's architecture is fundamentally different from Squad's:

- **Single agent, single context window** — one model instance handles all reasoning
- **Terminal-native** — runs in your shell, not in VS Code
- **Sandbox-first** — executes code in isolated environments with configurable approval policies
- **No persistent memory between sessions** — each invocation starts fresh (unless you use `AGENTS.md` or project-level instructions)
- **Model-agnostic within OpenAI** — switch between GPT-5.4, GPT-5.4-mini, GPT-5.4-nano via profiles

## The Architectural Comparison

| Dimension | Squad | Codex CLI |
|-----------|-------|-----------|
| **Agent count** | Multiple specialists (typically 5-6) | Single agent |
| **Execution model** | Parallel — agents work simultaneously | Sequential — one context handles all tasks |
| **Context strategy** | Replicated — each agent gets full repo context | Shared — one window manages everything |
| **Memory** | Persistent — charters, history, decisions committed to Git | Ephemeral — starts fresh each session |
| **Platform** | VS Code + GitHub Copilot | Terminal (any OS) |
| **Underlying model** | GitHub Copilot models | OpenAI API (GPT-5.4 family) |
| **Sandbox** | Git worktrees for parallel agent work | Docker/seatbelt/network-disabled containers |
| **Review enforcement** | Built-in — Tester agent can reject, different agent fixes | Manual — you review the output |
| **Autonomous mode** | Ralph (watch mode) — polls for issues, dispatches agents overnight[^2] | `--full-auto` flag — auto-approves all actions |
| **Installation** | `npm install -g @bradygaster/squad-cli && squad init` | `npm install -g @openai/codex` |
| **Open source** | Yes (MIT) | Yes (Apache 2.0) |
| **Repository footprint** | `.squad/` directory (multiple files, committed) | Optional `AGENTS.md` file |

## Where Squad Excels

### 1. Parallel Execution on Large Tasks

Squad's most compelling advantage is **concurrent agent work**. When a task touches frontend, backend, and tests simultaneously, Squad can dispatch three agents that work in parallel across isolated git worktrees[^2]. Codex CLI processes these sequentially — one context window, one task at a time.

In the documented Azure CLI case study, Jordan Selig used Squad to process **78 customer-reported GitHub issues in a single day**, deploying six agents (Lead, Investigator, Reproducer, Analyst, Fixer, Scribe). The Reproducer agent provisioned real Azure infrastructure to validate fixes. Multiple Fixer agents worked simultaneously via isolated worktrees. Result: 46 issues addressed, 8 consolidated PRs, 80+ new unit tests[^4].

This is a workflow that would take weeks with a single-agent tool — not because the agent is slow, but because the tasks are independent and benefit from parallel reasoning.

### 2. Persistent Team Memory

Squad's `.squad/` directory accumulates institutional knowledge over time. Agent history files record what each specialist has learned about your project. The `decisions.md` file captures architectural decisions with rationale. Skills files compress learnings from past work into reusable patterns[^1][^2].

Codex CLI has no equivalent. Each session starts with a blank context (plus whatever you put in `AGENTS.md`). You can partially bridge this gap with detailed `AGENTS.md` files and project-level instructions, but Squad's memory is automatic, structured, and versioned.

### 3. Built-In Review Protocols

Squad's Tester agent has explicit authority to reject work[^2]. When it does, a **different** agent provides the fix — not the original author. This cross-agent review catches blind spots that self-review misses.

Codex CLI relies entirely on human review. You can run Codex CLI in `suggest` mode (where it proposes changes for your approval) or `full-auto` mode (where it executes without asking), but there is no built-in mechanism for one agent to review another agent's work.

### 4. Autonomous Watch Mode

Squad's Ralph agent can poll for new issues, triage them, and dispatch agents autonomously — including overnight scheduling with configurable start and end times[^2]. This turns Squad into an always-on development team that processes your backlog while you sleep.

Codex CLI has no equivalent autonomous polling mode. You can build similar workflows with external tooling (cron jobs, GitHub Actions), but it requires custom scaffolding.

## Where Codex CLI Excels

### 1. Simplicity and Speed

Codex CLI is a single command: `codex "fix the login bug"`. There is no team to configure, no routing rules to write, no agent charters to maintain. For the vast majority of development tasks — fixing a bug, writing a function, refactoring a module — a single capable agent is all you need.

Squad's `.squad/` directory introduces meaningful cognitive overhead. You need to understand agent roles, routing policies, casting systems, and ceremony configurations. This overhead pays off for large, parallelisable tasks but is friction for small ones.

### 2. Terminal-Native Workflow

Codex CLI runs wherever you have a terminal — SSH sessions, CI/CD pipelines, headless servers, remote machines. It does not require VS Code or GitHub Copilot. This makes it the natural choice for automation, scripting, and integration into existing command-line workflows.

Squad is tightly coupled to GitHub Copilot and launches via `copilot --agent squad`[^1]. It lives in the VS Code ecosystem. If your workflow is terminal-first, Codex CLI fits; if it is VS Code-first, Squad fits.

### 3. Model Flexibility and Cost Control

Codex CLI lets you choose any OpenAI model (GPT-5.4, mini, nano, Codex-specific) and switch between them per task using profiles. This provides granular cost control — use nano for simple tasks, mini for most work, full 5.4 for complex reasoning.

Squad uses whatever model GitHub Copilot provides. You have less control over model selection, and the cost is absorbed into your Copilot subscription rather than billed per-token. This is simpler but less transparent.

### 4. Sandbox Security

Codex CLI executes code inside network-disabled sandboxed containers with configurable approval policies[^3]. You can run it in `suggest` mode (every action requires approval), `auto-edit` mode (file edits auto-approved, commands require approval), or `full-auto` mode (everything auto-approved).

Squad relies on git worktrees for isolation — agents work in separate working trees, not sandboxed containers. The security model is different: Squad trusts the agents within the Copilot environment, while Codex CLI defaults to distrust and requires explicit permission escalation.

## The Deeper Comparison: Orchestration vs Capability

The fundamental difference is not about features — it is about **what each tool optimises for**.

**Codex CLI optimises for agent capability.** Give a single, powerful agent the best possible model, the most context, and the most tools, then let it solve the problem. The bet is that one excellent agent with a large context window can handle most development tasks.

**Squad optimises for agent orchestration.** Take multiple agents, give each a specialised role and persistent memory, enforce review protocols between them, and coordinate their work. The bet is that multi-agent coordination produces better outcomes than a single agent, especially on large or complex tasks.

These are not mutually exclusive bets. In fact, they point to a likely convergence:

- Codex CLI is moving toward multi-agent patterns. The `codex-exec` format already supports multi-step workflows, and the community is building orchestration layers (OMX, Gas Town, NanoClaw) on top of Codex CLI.
- Squad is, at its core, an orchestration layer that could theoretically coordinate any agent — including Codex CLI instances — rather than being permanently coupled to GitHub Copilot.

## When to Use Which

| Scenario | Use Squad | Use Codex CLI |
|----------|-----------|---------------|
| Large feature spanning frontend + backend + tests | ✅ Parallel agents | ❌ Sequential |
| Quick bug fix or single-file change | ❌ Overhead not justified | ✅ One command |
| Overnight backlog triage | ✅ Ralph watch mode | ❌ No built-in polling |
| CI/CD pipeline integration | ❌ VS Code dependency | ✅ Terminal-native |
| Team with shared project knowledge | ✅ Persistent memory | ❌ Ephemeral sessions |
| Cost-sensitive API usage | ❌ Less model control | ✅ Per-token, model-switchable |
| Remote/headless server | ❌ Needs VS Code | ✅ SSH-friendly |
| Code review with fresh perspective | ✅ Cross-agent review | ❌ Self-review only |

## The Bigger Picture

Squad represents one answer to the question: **what comes after the single-agent coding assistant?** Instead of making one agent smarter, make a team of agents that specialise, remember, and review each other's work. The `.squad/` directory — committed to Git, travelling with the code — is an elegant solution to the memory problem that plagues all current coding agents.

Codex CLI represents a different answer: **make the single agent as capable and flexible as possible**, then let users compose higher-level workflows around it. The terminal-first, model-flexible, sandbox-secure approach prioritises developer control and integration over built-in orchestration.

Both approaches have real merit. The Azure CLI case study — 78 issues in a day with six parallel agents — demonstrates that multi-agent orchestration can unlock throughput that single-agent tools cannot match. But for the 90% of development work that involves focused, sequential problem-solving, a single powerful agent with no setup overhead remains hard to beat.

The interesting question is not which tool wins, but whether these architectures converge — and how quickly.

---

## Citations

[^1]: Squad Documentation — Brady Gaster. Installation, `.squad/` directory structure, agent charters, routing, CLI commands. <https://bradygaster.github.io/squad/>

[^2]: How Squad Runs Coordinated AI Agents Inside Your Repository — GitHub Blog. Context replication, parallel execution, review enforcement, Ralph watch mode, worktree spawning, thematic casting. <https://github.blog/ai-and-ml/github-copilot/how-squad-runs-coordinated-ai-agents-inside-your-repository/>

[^3]: Codex CLI — OpenAI. Terminal-based coding agent, sandbox architecture, approval policies, model selection. <https://github.com/openai/codex>

[^4]: How I Used Squad to Tackle 78 Azure CLI Issues in a Day — Jordan Selig. Six-agent deployment, 78 issues triaged, 46 addressed, 8 PRs, 80+ tests, real Azure infrastructure provisioning. <https://jordan-selig.net/blog/squad-azure-cli-backlog/>
