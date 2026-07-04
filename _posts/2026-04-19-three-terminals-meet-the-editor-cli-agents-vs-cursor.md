---
title: "Three Terminals Meet the Editor: Codex CLI vs Claude Code vs Gemini CLI vs Cursor"
date: 2026-04-19T00:00:00+00:00
last_modified_at: 2026-07-05T00:14:00+01:00
categories: [codex-cli, claude-code, gemini-cli, cursor, comparison, ide, terminal-agents, customisation]
description: "Cursor 3, released on 2 April 2026, ships parallel cloud agents, a visual design mode, and a tabbed agent window for managing local, cloud."
parent: "Articles"
nav_order: 928
---

![Sketchnote diagram for: Three Terminals Meet the Editor: Codex CLI vs Claude Code vs Gemini CLI vs Cursor](/sketchnotes/articles/2026-04-19-three-terminals-meet-the-editor-cli-agents-vs-cursor.png)

# Three Terminals Meet the Editor: Codex CLI vs Claude Code vs Gemini CLI vs Cursor

Cursor 3, released on 2 April 2026, ships parallel cloud agents, a visual design mode, and a tabbed agent window for managing local, cloud, and SSH instances simultaneously. It is not a terminal agent. It is an AI-native IDE built on the premise that most developers already live in VS Code, and the better move is to embed AI there rather than ask them to shift to a terminal.

[Three Terminals, Three Fates](../premium-articles/09-three-terminals-three-fates.md) compared Codex CLI, Claude Code, and Gemini CLI across the opposite premise: that the terminal is sufficient. This article adds Cursor to that comparison and examines all four tools across two dimensions: **what they can do** (functionality) and **how you can shape them** (customisation). The three terminals share more DNA with each other than any of them shares with Cursor, but Cursor fills gaps that no terminal agent addresses.

### Versions compared

This comparison is pinned to the following versions, all current as of 19 April 2026:

| Tool | Version | Release date |
|------|---------|-------------|
| **Codex CLI** | v0.121.0 (stable) / v0.122.0-alpha.11 | April 2026 |
| **Claude Code** | v2.1.114 | 17 April 2026 |
| **Gemini CLI** | v0.38.2 (stable) / v0.39.0-preview.0 | April 2026 |
| **Cursor** | 3.1 | 13 April 2026 |

All four tools release frequently; features described here may change within weeks. Where a feature is specific to an alpha, preview, or nightly build, this is noted.

## The philosophical split

The three CLI agents agree on one fundamental point: the terminal is the right interface for AI-assisted coding. Code goes in, code comes out, everything is scriptable and composable. They disagree on personality, with Codex CLI as the executor, Claude Code as the reasoner, and Gemini CLI as the explorer, but they share the same interface paradigm.

Cursor makes the opposite bet: the IDE is the right interface. Most developers already live in VS Code. Rather than asking them to context-switch to a terminal, Cursor embeds AI directly where they work. Tab completions predict your next edit. Inline diffs show changes before you accept them. The agent writes code in the same panes where you read it.

```mermaid
graph LR
    subgraph Terminal["Terminal-Native (Scriptable, Composable)"]
        CX["Codex CLI\nThe Executor"]
        CC["Claude Code\nThe Reasoner"]
        GC["Gemini CLI\nThe Explorer"]
    end

    subgraph IDE["IDE-Native (Visual, Integrated)"]
        CU["Cursor\nThe Editor"]
    end

    Terminal ---|"Shared: CLI interface\nMCP, AGENTS.md\nSandboxed execution"| IDE
```

Neither side is wrong. The question is which trade-offs matter for your workflow.

## Functionality comparison

### Core capabilities

| Feature | Codex CLI | Claude Code | Gemini CLI | Cursor |
|---------|-----------|-------------|------------|--------|
| **Interface** | Terminal | Terminal | Terminal | VS Code fork (GUI) |
| **Underlying models** | GPT-5.4 / GPT-5.3-Codex | Claude Opus 4.7 / Sonnet 4.6 | Gemini 3 Pro / Flash | Multi-model (GPT-5.4, Claude Opus 4.6, Gemini 3 Pro, Grok, proprietary Composer) |
| **Context window** | Configurable (GPT-5.4 supports 1M; default varies by model)[^15] | 1M (GA for Max/Team/Enterprise) | 1M (default) | Varies by model; Max Mode extends to model maximum (up to 1M with Claude Sonnet 1M)[^16] |
| **Open source** | Yes (Apache 2.0) | No | Yes (Apache 2.0) | No (proprietary, VS Code fork) |
| **Tab completion** | No | No | No | Yes (Cursor Tab, proprietary model) |
| **Inline editing** | No | No | No | Yes (Cmd+K, visual diffs) |
| **Multi-model switching** | No (OpenAI default; local models via `model_providers`) | No (Anthropic only) | No (Google only) | Yes (switch mid-session) |
| **MCP support** | Yes | Yes | Yes | Yes (5,000+ marketplace servers) |
| **AGENTS.md** | Native | Via fallback | Native | Supported |
| **Multimodal input** | Text, images (via `--image` flag)[^12] | Text, images | Text, images, PDFs, video | Text, images (paste/drag) |

The standout difference: **Cursor offers the broadest multi-model switching within a single session.** Codex CLI supports local models via `model_providers` (Ollama, llama.cpp) and custom endpoints, but cannot natively call Anthropic or Google APIs; you need a gateway or a different tool. Claude Code and Gemini CLI are locked to their parent company's models. Cursor lets you switch between GPT-5.4, Claude Opus 4.6, Gemini 3 Pro, and its own proprietary Composer model within a single session, though its Agent and Edit features are restricted to Cursor's custom models and cannot use BYOK keys.

The second: **Cursor is the only tool with tab completion.** The CLI agents do not predict your next edit as you type. They respond to prompts. Cursor Tab runs a specialised model that watches your edits and predicts multi-line completions in real time. This is a different category of assistance, one that augments typing rather than replacing it.

The third: **context window size favours the CLI agents at the high end.** Gemini CLI ships with one million tokens by default. Claude Code offers one million GA on Max/Team/Enterprise plans. Codex CLI's context window is configurable via `model_context_window` in config.toml, with GPT-5.4 supporting up to one million[^15]. Cursor's Max Mode extends the context to whatever the underlying model supports (up to one million with Claude Sonnet 1M), but standard mode uses smaller windows. All four tools can access large context in principle, but the CLI agents default to it while Cursor requires Max Mode[^16].

### Agent and automation capabilities

| Feature | Codex CLI | Claude Code | Gemini CLI | Cursor |
|---------|-----------|-------------|------------|--------|
| **Background/cloud agents** | Codex Cloud | Managed Agents + Routines (April 2026) | No | Cloud Agents (GA, up to 8 parallel) |
| **Parallel agents** | Subagents (TOML-defined) | Agent Teams (peer-to-peer) | `@agent` syntax (GA April 2026) | Agents Window (tiled layout, Cursor 3) |
| **CI/CD mode** | `codex exec` (first-class) | `claude -p` (headless) | `gemini -p` (headless) | Experimental `cursor agent -p` (headless, early preview)[^17] |
| **Headless execution** | Yes | Yes | Yes | No (requires GUI) |
| **Subagent architecture** | TOML-defined, parallel | Task tool, teams | 4 built-in + custom Markdown agents | Cloud agents with full desktop environments |
| **Design mode** | No | No | No | Yes (UI annotation via browser) |
| **Automations** | Via hooks | Via hooks | Via hooks | Webhook/CI-triggered (April 2026) |

These capabilities create real workflow trade-offs:

**CI/CD and scripting:** The CLI agents have a clear advantage. `codex exec`, `claude -p`, and `gemini -p` all run headlessly in pipelines. Cursor has introduced an experimental `cursor agent -p` headless mode, but it remains early preview and is not yet reliable enough for production CI/CD pipelines.[^17]

**Visual feedback:** Cursor wins decisively. Its Design Mode (Cursor 3) lets you annotate a browser-rendered UI and the agent sees your visual annotations. None of the CLI agents can do this.

**Cloud agents:** Both Cursor and Codex CLI offer mature cloud agent capabilities. Cursor's cloud agents give each agent a full desktop environment with browser, useful for front-end work where the agent needs to see what it built. Codex Cloud focuses on sandboxed code execution. Claude Code's Managed Agents and Routines, launched in April 2026, are newer but closing the gap.

### Security and sandboxing

| Dimension | Codex CLI | Claude Code | Gemini CLI | Cursor |
|-----------|-----------|-------------|------------|--------|
| **Isolation mechanism** | OS kernel (Seatbelt, Landlock, seccomp) | Application-layer hooks (26 events)[^14] | OS kernel + Docker/gVisor/LXC options | Application-layer approval + hooks + sandbox (Seatbelt/Landlock/seccomp) |
| **Network access** | Disabled by default | Configurable via hooks | Configurable per sandbox profile | Approval-gated |
| **Sandbox enabled by default** | Yes | Hooks require setup | No (opt-in via `-s` flag) | Yes (GA, agents stop 40 per cent less often) |
| **Self-hostable** | Yes (Apache 2.0) | No | Yes (Apache 2.0) | No |
| **`.cursorignore` / file exclusion** | Via sandbox policy | Via hooks | Via sandbox profiles | `.cursorignore` (files invisible to AI) |

Cursor added OS-level sandboxing in early 2026: Seatbelt on macOS, Landlock and seccomp on Linux, and WSL2-based isolation on Windows. This narrows the gap with Codex CLI and Gemini CLI. Cursor's sandbox, however, is focused on the agent's terminal commands; the IDE itself has full filesystem access. The CLI agents' sandbox applies to the entire execution environment.

Cursor's `.cursorignore` is a useful feature that the CLI agents lack in the same form. Files listed in `.cursorignore` are excluded from reading and codebase indexing, though Cursor's own documentation notes this is 'best-effort': files are not guaranteed to be blocked in all cases. For repositories with sensitive configuration files or proprietary code sections, it provides a reasonable layer of protection, but should not be the sole security control for truly sensitive files.

## Customisation comparison

Customisation is where the tools diverge most sharply. It determines how well each tool adapts to your codebase, your team's conventions, and your preferred workflow.

### Project instructions

| Feature | Codex CLI | Claude Code | Gemini CLI | Cursor |
|---------|-----------|-------------|------------|--------|
| **Project file** | AGENTS.md | CLAUDE.md (5-layer hierarchy) | GEMINI.md (configurable to read AGENTS.md) | `.cursor/rules/*.mdc` or `.cursorrules` (legacy) |
| **Hierarchical scoping** | Root-to-leaf, deeper overrides | Project + user level | Project > extension > global | File-glob scoping via MDC frontmatter |
| **Cross-tool portability** | AGENTS.md: 60,000+ repos, 25+ tools | CLAUDE.md: Claude-specific | GEMINI.md: Gemini-specific | `.cursorrules`: Cursor-specific; also reads AGENTS.md |
| **Global rules** | Config-level instructions | User-level CLAUDE.md | User-level GEMINI.md | Settings > 'Rules for AI' |
| **Team-managed rules** | Via repo | Via repo | Via repo | Dashboard-managed (Teams/Enterprise) |

Cursor's rule system is the most granular at the file level. MDC (Markdown with Context) files in `.cursor/rules/` support YAML frontmatter with `description`, `alwaysApply`, and `globs` properties:

```yaml
---
description: "React component conventions"
globs: "src/components/**/*.tsx"
alwaysApply: false
---

Use functional components with hooks.
Export named components, not default exports.
Co-locate tests in __tests__ subdirectories.
```

This means you can have different rules for your React components, your API routes, your test files, and your infrastructure code, all activated automatically based on which files the agent is working with. The CLI agents' hierarchical systems (deeper AGENTS.md files override shallower ones) achieve something similar but with directory-based rather than glob-based scoping.

The portability advantage goes to the CLI agents. AGENTS.md is an open standard supported by 25+ tools and present in 60,000+ repositories. `.cursorrules` files only work in Cursor.

### Model and provider configuration

| Feature | Codex CLI | Claude Code | Gemini CLI | Cursor |
|---------|-----------|-------------|------------|--------|
| **Model selection** | Config or CLI flag | Config or CLI flag | Config or CLI flag | Per-feature dropdown (chat, agent, tab) |
| **Per-model system prompts** | Yes (separate `.md` per model generation) | No | No | No |
| **BYOK (Bring Your Own Key)** | N/A (uses OpenAI account) | N/A (uses Anthropic account) | N/A (uses Google account) | Partially deprecated: works for chat, not for Agent/Edit |
| **Custom provider endpoints** | Yes (`model_providers` in config.toml) | No | No | Yes (OpenAI, Anthropic, Google, Azure, Bedrock) |
| **Local model support** | Yes (Ollama, llama.cpp via `model_providers`) | No | No | Limited (via BYOK API key to local endpoint) |

Codex CLI's local model support is more mature than Cursor's. You can configure multiple model providers in `config.toml` with profile switching:

```toml
[model_providers.gb10]
name = "GB10 Ollama"
base_url = "http://localhost:11434/v1"
wire_api = "responses"

[profiles.local-gemma]
model = "gemma4:31b"
model_provider = "gb10"
model_instructions_file = "~/prompts/minimal-codex.md"
```

Codex CLI is also unique in shipping **per-model system prompts**: separate instruction files optimised for each model generation. Compact variants for fine-tuned models save roughly two-thirds of the prompt text by not re-explaining tools the model already understands. The `model_instructions_file` config key lets you replace the system prompt entirely for local models, reducing overhead from roughly 8,500 to roughly 3,000 tokens.

Cursor's BYOK situation is complicated. It was announced for partial deprecation in late 2025. Agent and Edit features use Cursor's custom models and cannot route to external keys. Chat completions for standard models may still work with BYOK. This limits Cursor's flexibility for teams that want to control their model infrastructure.

### Hooks and programmable governance

| Feature | Codex CLI | Claude Code | Gemini CLI | Cursor |
|---------|-----------|-------------|------------|--------|
| **Hook events** | 5 (SessionStart, Stop, UserPromptSubmit, PreToolUse, PostToolUse) | 26 lifecycle events[^14] | 11 (BeforeTool, AfterTool, BeforeAgent, AfterAgent, BeforeModel, AfterModel, etc.) | 20 (18 agent hooks + 2 Tab hooks)[^13] |
| **Pre-execution validation** | PreToolUse (Bash only) | PreToolUse (all tools) | BeforeTool (all tools) | preToolUse, beforeShellExecution, beforeMCPExecution, beforeReadFile |
| **Post-execution audit** | PostToolUse (Bash only) | PostToolUse (all tools) | AfterTool + AfterAgent | postToolUse, afterShellExecution, afterMCPExecution, afterFileEdit |
| **Custom tool creation** | Via MCP servers | Via MCP servers | Via MCP servers + custom agents | Via MCP servers + VS Code extensions |
| **Security partner integrations** | Community-built | Community-built | Community-built | Semgrep, Snyk, Noma, MintMCP, Oasis, 1Password[^13] |

All four tools now have hook systems, though they differ in breadth. Claude Code leads with 26 lifecycle events across nine categories, covering subagent, compaction, worktree, and MCP events. Gemini CLI has 11 event types. Cursor, which added hooks in October 2025 (Cursor 1.7)[^13], has 7+ events focused on shell execution, MCP, file operations, and prompt submission. Codex CLI has the narrowest surface with five events, and PreToolUse/PostToolUse currently fire only for Bash tool calls.

For Stage 3 and Stage 4 teams in the adoption model, where agents run with minimal oversight and governance must be automated, the depth of the hook system matters. Claude Code's 26-event surface covers the widest range of governance scenarios. Cursor's hooks, while newer, have attracted security vendor integrations (Semgrep, Snyk, 1Password) that the CLI agents' hook ecosystems have not yet matched.

### Context and memory

| Feature | Codex CLI | Claude Code | Gemini CLI | Cursor |
|---------|-----------|-------------|------------|--------|
| **Codebase indexing** | No (reads files on demand) | No (reads files on demand) | No (reads files on demand) | Yes (embedding-based semantic search) |
| **`@` context references** | No | No | No | Yes (`@file`, `@codebase`, `@web`, `@docs`, `@git`, `@notepad`) |
| **Documentation indexing** | No | No | No | Yes (`@docs` with custom URLs) |
| **Memory system** | Yes (create, consolidate, clean, delete, GA April 2026) | `/memory` command | Gemini memory | Notepads (persistent text documents) |
| **Context compaction** | Automatic summarisation | Automatic summarisation | Automatic summarisation | Automatic summarisation |

Cursor's **codebase indexing** is a standout feature that no CLI agent replicates. When you type `@codebase` in Cursor, it runs a semantic search across your entire project using pre-computed embeddings. The CLI agents read files on demand; they can search with grep and glob, but they do not maintain a persistent semantic index.

The `@docs` feature is equally distinctive. Point Cursor at a documentation URL (React docs, your internal API docs, a framework guide) and it indexes the content for reference in conversations. None of the CLI agents offer this.

Cursor's **Notepads** serve a similar role to AGENTS.md sections but are managed through the Cursor UI rather than as files in the repository. You can reference them with `@notepad/name` in any conversation. This is convenient for personal context but not portable: Notepads do not live in version control.

## The pricing reality

| Tier | Codex CLI | Claude Code | Gemini CLI | Cursor |
|------|-----------|-------------|------------|--------|
| **Free** | — | — | 1,000 req/day | Hobby (limited) |
| **\$20/mo** | Plus | Pro | Gemini Advanced | Pro |
| **\$60/mo** | — | — | — | Pro+ (3x credits) |
| **\$100/mo** | Pro 5x | Max 5x | — | — |
| **\$200/mo** | Pro 20x | Max 20x | — | Ultra (20x usage) |
| **Team** | Enterprise (custom) | \$100/user/mo (Max plan) | Via Google Workspace | \$40/user/mo |
| **Billing model** | Subscription + API overflow | Subscription + API overflow | Free tier + API pay-as-you-go | Credit-based (monthly pool) |

Cursor's pricing is competitive at the individual level: \$20/mo Pro matches the CLI agents' entry tiers. The credit-based billing model, introduced in June 2025, means your monthly allowance depletes based on which model you use. Heavy use of Claude Opus 4.6 through Cursor burns credits faster than using Claude Sonnet.

For teams, Cursor at \$40/user/mo is cheaper than Claude Code's Team/Max pricing at \$100/user/mo, but more expensive than the Codex Plus and Gemini Free combination at \$20/user/mo.

## When Cursor wins

Cursor is the right choice when:

1. **Your team lives in VS Code.** The migration cost from VS Code to Cursor is near zero: your extensions, keybindings, and settings carry over. The migration cost from VS Code to a terminal agent is a workflow paradigm shift.

2. **You need tab completion.** If predictive inline editing is a core part of your workflow, no CLI agent offers this. Cursor Tab is useful and has no terminal equivalent.

3. **You want multi-model flexibility without switching tools.** Need Claude for reasoning, GPT for execution, Gemini for exploration, all in one window? Cursor is the only single tool that does this.

4. **Visual diff review matters.** Cursor shows changes as inline diffs in the editor. CLI agents show changes as text output. For reviewing complex multi-file edits, the visual presentation is faster to parse.

5. **Design Mode is relevant.** If you do front-end work and want to annotate a rendered UI for the agent to see, Cursor's Design Mode is unique.

6. **You need codebase-wide semantic search.** The `@codebase` embedding index finds semantically related code that grep would miss. CLI agents have no equivalent.

## When the CLI agents win

The terminal agents are the right choice when:

1. **CI/CD integration is required.** `codex exec`, `claude -p`, and `gemini -p` run headlessly in pipelines. Cursor's experimental `cursor agent -p` is not yet production-ready.

2. **Deep programmable governance is needed.** Claude Code's 26-event hook system covers subagents, compaction, worktrees, and MCP, the widest governance surface. Cursor has hooks (7+ events) but a narrower surface than the CLI leaders. Codex CLI's five events are the most limited.

3. **Context window size matters.** One million tokens (CLI agents) versus roughly 70,000 to 120,000 usable (Cursor standard mode). For large codebase analysis or long sessions, the CLI agents have eight to 14 times more working memory.

4. **Local model support is needed.** Codex CLI's `model_providers` configuration with profile switching and custom system prompts is mature. Cursor's local model support is limited.

5. **Self-hosting or auditability is required.** Codex CLI and Gemini CLI are Apache 2.0 open source. Cursor is proprietary.

6. **You are building agentic infrastructure.** The CLI agents are composable building blocks: pipe them together, wrap them in scripts, orchestrate them with tools like NanoClaw or `ccb`. Cursor is a monolithic application.

7. **Token efficiency matters.** Codex CLI uses roughly four times fewer tokens than Claude Code for comparable tasks. Cursor's token consumption data is limited, but IDE overhead (codebase indexing queries, context assembly) adds to the total.

## The hybrid pattern

Running both is often the most effective approach:

```mermaid
graph TD
    subgraph IDE["Cursor (Daily Editing)"]
        TAB["Tab Completion\n(while typing)"]
        INLINE["Inline Edits\n(Cmd+K)"]
        REVIEW["Visual Diff Review"]
        DESIGN["Design Mode\n(front-end)"]
    end

    subgraph Terminal["CLI Agents (Heavy Lifting)"]
        EXEC["Codex CLI\n(batch execution, CI/CD)"]
        REASON["Claude Code\n(architecture, debugging)"]
        EXPLORE["Gemini CLI\n(exploration, free tier)"]
    end

    subgraph Shared["Shared Layer"]
        MCP["MCP Servers"]
        AGENTS["AGENTS.md"]
        SKILLS["SKILL.md"]
    end

    IDE --> Shared
    Terminal --> Shared
```

Use Cursor for moment-to-moment editing: tab completions, inline changes, visual review. Switch to the terminal for heavier agent tasks: batch refactoring with Codex, architectural reasoning with Claude Code, codebase exploration with Gemini CLI. The shared MCP servers, AGENTS.md files, and skills work across all four tools.

The pattern is already established. Many of the most productive developers run Cursor alongside a CLI agent, typically Claude Code. The IDE handles the 80 per cent of work that benefits from visual feedback; the terminal handles the 20 per cent that benefits from deep context, scriptability, and governance.

## The verdict

Cursor and the CLI agents are not competing for the same niche. They are competing for different phases of the development workflow:

| Phase | Best tool | Why |
|-------|-----------|-----|
| **Typing code** | Cursor | Tab completion, inline predictions |
| **Small edits** | Cursor | Cmd+K, visual inline diffs |
| **Exploring unfamiliar code** | Gemini CLI or Cursor | Gemini: 1M context, free. Cursor: `@codebase` semantic search |
| **Architectural reasoning** | Claude Code | Deepest reasoning, largest effective context |
| **Batch execution** | Codex CLI | Token efficient, kernel sandbox, subagent parallelism |
| **CI/CD pipelines** | Codex CLI or Gemini CLI | Headless execution, scriptable |
| **Front-end design iteration** | Cursor | Design Mode, visual browser annotation |
| **Enterprise governance** | Claude Code (26 hooks) or Cursor (security partner integrations) | Deepest hook surface + self-hosting (CLI) or vendor integrations (Cursor) |
| **Cost-constrained exploration** | Gemini CLI | 1,000 free requests/day |
| **Running local models** | Codex CLI | Mature provider config, custom system prompts, profile switching |

The three terminals and the editor are not three fates plus one. They are four tools for four different moments in a developer's day. The convergence thesis from article 09 still holds: MCP, AGENTS.md, and the four core tools (Read, Search, Edit, Execute) are becoming universal infrastructure. But Cursor adds capabilities that no terminal can replicate (tab completion, semantic indexing, visual diffs, Design Mode), just as the terminals add capabilities that Cursor cannot match (headless CI/CD, one-million-token context, full scriptability, self-hosting).

Invest in the shared layer. Use the right tool for the moment. The editor and the terminal are not rivals; they are complements.

*This article extends the analysis from [Three Terminals, Three Fates](../premium-articles/09-three-terminals-three-fates.md) (premium article 09 in the From Experiment to Enterprise series). For system prompt architecture details across the CLI agents and Pi, see [The DNA of Coding Agents](2026-04-19-system-prompts-compared-codex-gemini-claude-code.md).*

[^12]: Codex CLI image input: `--image` (or `-i`) flag accepts PNG, JPEG, GIF, and WebP files. Full-resolution image inspection and `view_image` tool stable since March 2026. https://developers.openai.com/codex/cli/features
[^13]: Cursor Hooks, added in Cursor 1.7 (October 2025). 20 hook events: 18 agent hooks (`sessionStart`, `sessionEnd`, `preToolUse`, `postToolUse`, `postToolUseFailure`, `subagentStart`, `subagentStop`, `beforeShellExecution`, `afterShellExecution`, `beforeMCPExecution`, `afterMCPExecution`, `beforeReadFile`, `afterFileEdit`, `beforeSubmitPrompt`, `preCompact`, `stop`, `afterAgentResponse`, `afterAgentThought`) + 2 Tab hooks (`beforeTabFileRead`, `afterTabFileEdit`). Security partner integrations: Semgrep, Snyk, Noma, MintMCP, Oasis, 1Password. https://cursor.com/docs/hooks and https://cursor.com/blog/hooks-partners
[^14]: Claude Code hooks: 26 lifecycle events across 10 categories (Session, User Input, Tool Execution, Agent/Team, Conversation Flow, Config/Environment, Worktree, Context Management, Notifications, MCP). https://code.claude.com/docs/en/hooks
[^15]: Codex CLI context window: configurable via `model_context_window` in `config.toml`. GPT-5.4 supports up to 1M tokens; no documented default value. https://developers.openai.com/codex/cli/configuration
[^16]: Cursor context window: varies by model selected. Max Mode extends the context window to the model's maximum (up to 1M tokens with Claude Sonnet 1M). Standard mode uses smaller windows. https://cursor.com/docs/context
[^17]: Cursor headless mode: experimental `cursor agent -p` command for running agents without the GUI. Early preview, not yet recommended for production CI/CD. https://cursor.com/changelog
