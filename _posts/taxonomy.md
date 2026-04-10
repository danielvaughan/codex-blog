---
title: "Article Taxonomy"
---

# Article Tag Taxonomy — Proposal (2026-04-08)

> Status: APPLIED — all article files have been updated with these tags.

This document defines a semi-controlled taxonomy for tagging every article in `/articles/`. The goal is to make articles discoverable by filtering and search, while keeping the vocabulary small enough that tags remain meaningful.

---

## How It Works

1. Every article gets **exactly one primary category** (mutually exclusive).
2. Every article gets **2-5 secondary tags** from the approved list below.
3. No free-form tags — all tags must appear in this file first.
4. To propose a new tag, add it here in a PR and get it reviewed before using it.

---

## 1. Primary Categories

A primary category describes *what kind of article this is*. Every article gets exactly one.

| Category | Description |
|---|---|
| `getting-started` | Installation, first-run, onboarding, learning paths. The reader is new to Codex CLI or to a specific feature. |
| `configuration` | Reference material for config files, flags, profiles, feature flags, TUI tuning. "How do I set X up?" |
| `architecture` | Internals, design decisions, how things work under the hood (agent loop, sandbox, compaction, Rust rewrite, app-server protocol). |
| `security` | Sandbox hardening, network policies, approval frameworks, credential management, vulnerability scanning, compliance (HIPAA/SOC2/FedRAMP). |
| `orchestration` | Multi-agent patterns, subagents, agentic pods, symphony/harness-engineering, workflow orchestration, plan-review loops. |
| `workflow-patterns` | Practical development patterns: TDD with agents, spec-driven dev, code review, prompting strategies, mid-turn steering, planning mode. |
| `competitive-landscape` | Tool comparisons, migration guides, community sentiment, market positioning. Codex vs X articles. |
| `ecosystem` | Plugins, skills, MCP servers, SDKs, IDE extensions, third-party integrations (Vercel, Figma, Slack, Linear, PagerDuty, dbt, Neo4j). |
| `ci-cd` | CI/CD pipelines, `codex exec`, GitHub Actions, GitLab CI, non-interactive mode, structured output, automations. |
| `models` | Model selection, model lineage, Spark/Max variants, reasoning effort, cost/token economics, billing. |
| `opinion` | Editorials, think-pieces, strategic analysis, community sentiment roundups, philosophy of agentic engineering. |
| `reference` | Command references, configuration references, JSONL schemas, API specs, complete guides meant to be bookmarked. |
| `language-guide` | Language- or framework-specific guides (Python, Go, Rust, Java, .NET, mobile/iOS/Android, frontend/React). |
| `cloud` | Codex Cloud, remote sessions, background tasks, desktop app, CC Pocket, cross-surface sync, four-surface architecture. |

---

## 2. Approved Secondary Tags

Secondary tags describe *what the article is about*. Grouped by theme.

### Core Concepts
| Tag | Description |
|---|---|
| `agents-md` | AGENTS.md files — creation, patterns, bloat, monorepo strategies, open standard |
| `skills` | SKILL.md files, skills ecosystem, agentskills.io, progressive disclosure |
| `plugins` | Plugin system, plugin manifest, distribution, marketplace, installed-by-default |
| `mcp` | Model Context Protocol — MCP servers, MCP integration, bidirectional bridges |
| `hooks` | Lifecycle hooks — PreToolUse, PostToolUse, SessionStart, SessionStop |
| `config-toml` | config.toml settings, profiles, feature flags, inline overrides |
| `approval-modes` | Suggest, auto-edit, full-auto, approval fatigue, Starlark rules engine |

### Agent Architecture
| Tag | Description |
|---|---|
| `agent-loop` | How the agentic loop works internally, responses API, phases |
| `subagents` | Subagent spawning, path addressing, delegation, multi-agent v2 |
| `context-management` | Context window, compaction, token limits, intelligent forgetting |
| `sandbox` | Kernel-level sandboxing — Landlock, seccomp, Seatbelt, Docker isolation |
| `codex-rs` | The Rust rewrite, wire protocol, ratatui, zero-dependency architecture |
| `app-server` | JSON-RPC protocol, WebSocket transport, remote deployment, headless mode |

### Models and Cost
| Tag | Description |
|---|---|
| `model-selection` | Choosing between models, model routing, tiered inference |
| `codex-spark` | GPT-5.3-Codex-Spark, Cerebras inference, real-time coding |
| `cost-optimization` | Token strategy, credit economy, quota, billing, cost calculators |
| `model-lifecycle` | Deprecations, migrations, lineage, GPT-5.x transition |

### Workflow and Productivity
| Tag | Description |
|---|---|
| `planning` | Plan mode, spec-driven development, plan-review loops |
| `code-review` | PR review, /review command, cross-model review, adversarial review |
| `testing` | TDD, evals, LongCLI-Bench, long-horizon benchmarks, quality gates |
| `prompting` | Prompt engineering, effective prompting strategies |
| `git-integration` | Commit attribution, worktrees, parallel development |
| `automation` | Scheduled tasks, triggers, event-driven workflows, background agents |
| `voice-input` | Voice transcription, Whisper, hands-free workflows, realtime sessions |
| `non-coding` | File automation, image workflows, toolkit expansion, general-purpose agent use |

### Platform and Deployment
| Tag | Description |
|---|---|
| `github-actions` | GitHub Actions integration, the official Codex action, agentic workflows |
| `gitlab` | GitLab CI/CD, Duo Agent Platform |
| `windows` | Windows support, WSL, native sandbox, Microsoft Store |
| `docker` | Containerised environments, codex-universal image |
| `kubernetes` | K8s, Helm, cloud-native, Agent Sandbox CRD |
| `codex-cloud` | Cloud execution, remote sessions, best-of-N attempts |
| `desktop-app` | Codex Desktop, theming, automations, review queue |
| `mobile` | CC Pocket, iOS, Android, Xcode workflows |
| `ide-extension` | VS Code, JetBrains integration, hybrid workflows |

### Ecosystem and Integrations
| Tag | Description |
|---|---|
| `typescript-sdk` | TypeScript SDK, programmatic embedding |
| `python-sdk` | Python SDK, app-server RPC |
| `agents-sdk` | OpenAI Agents SDK, cookbook patterns |
| `observability` | OpenTelemetry, metrics, tracing, session analytics, JSONL rollout |
| `memory` | Persistent memory, knowledge graphs, Graphiti, cross-session context |
| `third-party` | Vercel, Figma, Slack, Linear, Notion, Google Drive, PagerDuty, Datadog, dbt, Neo4j |
| `open-source` | OSS fund, community, contribution crisis, MIT license |

### Security and Compliance
| Tag | Description |
|---|---|
| `network-security` | Domain allowlists, requirements.toml, air-gapped deployments |
| `credential-management` | OAuth, API keys, device code flow, bearer tokens |
| `compliance` | HIPAA, SOC2, FedRAMP, regulated environments, audit trails |
| `vulnerability-scanning` | Codex Security agent, Aardvark, threat modelling, patch generation |

### Competitive and Strategic
| Tag | Description |
|---|---|
| `claude-code` | Anthropic's Claude Code — comparison, migration, interop |
| `cursor` | Cursor IDE — comparison, migration |
| `github-copilot` | GitHub Copilot — comparison, migration, coding agent |
| `competitor-tools` | Other tools: Goose, Cline, Amp, Antigravity, Kiro, OpenCode, Pi, Devin |
| `community-sentiment` | Reddit roundups, community opinions, adoption signals |
| `product-direction` | Roadmap speculation, strategic pivots, superapp thesis |

### Personas (cross-cutting — use when an article clearly maps to a persona's needs)
| Tag | Description |
|---|---|
| `enterprise` | Enterprise deployment, RBAC, managed policies, team configuration |
| `team-workflow` | Multi-person team patterns, engineering management, agentic pods |
| `incident-response` | On-call automation, PagerDuty, Datadog, production safety |

---

## 3. Migration Table: Current Tags to New Tags

This table maps every tag currently in use to its new home. Tags are merged, renamed, or dropped.

| Current Tag | Action | New Tag(s) | Notes |
|---|---|---|---|
| `aardvark` | Merge | `vulnerability-scanning` | Product name — too specific for a tag |
| `accountability` | Drop | — | Covered by article content, not a filterable concept |
| `admin` | Merge | `enterprise` | |
| `advanced` | Drop | — | Not a useful filter; use primary category instead |
| `adversarial` | Merge | `code-review` | Covered by "cross-model adversarial review" in article |
| `agent-failures` | Drop | — | Covered by primary category `workflow-patterns` or `reference` |
| `agent-hq` | Drop | — | GitHub Copilot product feature, too specific |
| `agent-loop` | Keep | `agent-loop` | |
| `agents` | Merge | `agents-md` or `subagents` | Ambiguous — assign per article |
| `agents-md` | Keep | `agents-md` | |
| `AGENTS.md` | Merge | `agents-md` | Normalise casing |
| `agents-sdk` | Keep | `agents-sdk` | |
| `agentic` | Drop | — | Too generic; everything is agentic |
| `agentic-coding` | Drop | — | Too generic |
| `agentic-platform` | Merge | `product-direction` | |
| `agentic-pod` | Merge | `team-workflow` | Agentic pod is a team pattern |
| `aider` | Merge | `competitor-tools` | |
| `android` | Merge | `mobile` | |
| `api-credits` | Merge | `cost-optimization` | |
| `api-errors` | Drop | — | Covered by troubleshooting content |
| `app-server` | Keep | `app-server` | |
| `appsec` | Merge | `vulnerability-scanning` | |
| `approval-fatigue` | Merge | `approval-modes` | |
| `architecture` | Drop | — | Use primary category instead |
| `audit-trail` | Merge | `compliance` | |
| `automation` | Keep | `automation` | |
| `automations` | Merge | `automation` | Normalise to singular |
| `background-agents` | Merge | `automation` | |
| `background-tasks` | Merge | `automation` | |
| `background-terminal` | Drop | — | Experimental feature, too specific |
| `batch-approval` | Merge | `approval-modes` | |
| `bearer-token` | Merge | `credential-management` | |
| `benchmarks` | Merge | `testing` | |
| `best-practices` | Drop | — | Too generic; use primary category |
| `caliber` | Drop | — | Third-party tool name, too specific |
| `cc-pocket` | Merge | `mobile` | Product surface name |
| `cc-switch` | Drop | — | Third-party tool, too specific |
| `cerebras` | Merge | `codex-spark` | Hardware partner, covered by codex-spark |
| `changelog` | Merge | `product-direction` | |
| `chatgpt-pro` | Drop | — | Pricing tier, too specific |
| `ci` | Merge | `github-actions` or primary `ci-cd` | Per article |
| `ci-cd` | Keep as primary | Primary: `ci-cd` | |
| `cicd` | Merge | Primary: `ci-cd` | Normalise spelling |
| `claude-code` | Keep | `claude-code` | |
| `cloud` | Merge | `codex-cloud` | |
| `co-authored-by` | Merge | `git-integration` | |
| `code-mode` | Drop | — | UI detail, too specific |
| `code-review` | Keep | `code-review` | |
| `codex` | Drop | — | Redundant; everything is about Codex |
| `codex-app` | Merge | `desktop-app` | |
| `codex-cli` | Drop | — | Redundant; every article is about Codex CLI |
| `codex-cloud` | Keep | `codex-cloud` | |
| `codex-exec` | Keep as part of primary `ci-cd` | — | Covered by primary category |
| `codex-integration` | Drop | — | Too generic |
| `codex-rs` | Keep | `codex-rs` | |
| `codex-security` | Merge | `vulnerability-scanning` | |
| `codex-spark` | Keep | `codex-spark` | |
| `coding-agent` | Drop | — | Too generic |
| `commit-attribution` | Merge | `git-integration` | |
| `communication-style` | Drop | — | Covered by article content |
| `community` | Merge | `open-source` | |
| `community-ecosystem` | Merge | `open-source` | |
| `compaction` | Merge | `context-management` | |
| `comparison` | Drop | — | Use primary category `competitive-landscape` |
| `competitive-landscape` | Keep as primary | Primary: `competitive-landscape` | |
| `compliance` | Keep | `compliance` | |
| `compound-engineering` | Merge | `team-workflow` | Concept covered in article |
| `config` | Merge | `config-toml` | |
| `config-management` | Merge | `config-toml` | |
| `config-toml` | Keep | `config-toml` | |
| `configuration` | Merge | `config-toml` or primary `configuration` | Per article |
| `context` | Merge | `context-management` | |
| `context-architect` | Drop | — | Role name, too specific |
| `context-compaction` | Merge | `context-management` | |
| `context-management` | Keep | `context-management` | |
| `context-rot` | Merge | `context-management` | |
| `context7` | Drop | — | Specific MCP server, too specific |
| `cost` | Merge | `cost-optimization` | |
| `cost-management` | Merge | `cost-optimization` | |
| `cross-model` | Merge | `code-review` | Used in context of cross-model review |
| `cursor` | Keep | `cursor` | |
| `datadog` | Merge | `third-party` | |
| `debugging` | Drop | — | Use primary category |
| `dev-server` | Drop | — | Feature detail, too specific |
| `devin` | Merge | `competitor-tools` | |
| `dictation` | Merge | `voice-input` | |
| `ecosystem` | Drop | — | Use primary category |
| `enterprise` | Keep | `enterprise` | |
| `evals` | Merge | `testing` | |
| `evaluation` | Merge | `testing` | |
| `exec` | Drop | — | Covered by primary `ci-cd` |
| `execpolicy` | Merge | `approval-modes` | |
| `execution` | Drop | — | Too generic |
| `experimental` | Drop | — | Temporal; features graduate out of experimental |
| `features` | Drop | — | Too generic |
| `features-table` | Drop | — | Implementation detail |
| `feedback-loop` | Drop | — | Conceptual; covered by article |
| `figma` | Merge | `third-party` | |
| `fork` | Drop | — | CLI subcommand, too specific |
| `friendly` | Drop | — | Personality trait, not a tag |
| `general-purpose-agent` | Merge | `non-coding` | |
| `git` | Merge | `git-integration` | |
| `github-actions` | Keep | `github-actions` | |
| `github-agentic-workflows` | Merge | `github-actions` | |
| `github-copilot` | Keep | `github-copilot` | |
| `gmail` | Merge | `third-party` | |
| `google-antigravity` | Merge | `competitor-tools` | |
| `google-drive` | Merge | `third-party` | |
| `governance` | Merge | `compliance` | |
| `gpt-5-2-codex` | Merge | `model-lifecycle` | Specific model version |
| `gpt-5-3-codex` | Merge | `model-lifecycle` | Specific model version |
| `gpt-5-3-codex-spark` | Merge | `codex-spark` | |
| `graphiti` | Merge | `memory` | |
| `hands-free` | Merge | `voice-input` | |
| `harness-engineering` | Merge | `team-workflow` | Harness engineering is a workflow concept |
| `health-checks` | Drop | — | Implementation detail |
| `history` | Drop | — | Use primary category `getting-started` or `opinion` |
| `hooks` | Keep | `hooks` | |
| `human-in-the-loop` | Drop | — | Conceptual; covered by `approval-modes` |
| `hybrid` | Drop | — | Too vague |
| `image-generation` | Merge | `non-coding` | |
| `images` | Merge | `non-coding` | |
| `implementer` | Drop | — | Agent role, too specific |
| `incident-response` | Keep | `incident-response` | |
| `installation` | Drop | — | Use primary category `getting-started` |
| `installed-by-default` | Drop | — | Plugin detail, too specific |
| `integration` | Drop | — | Too generic |
| `intent-driven` | Drop | — | Conceptual |
| `interactive-refinement` | Drop | — | Conceptual |
| `internals` | Drop | — | Use primary category `architecture` |
| `ios` | Merge | `mobile` | |
| `issue-assignment` | Drop | — | Feature detail |
| `json-rpc` | Merge | `app-server` | |
| `kiro` | Merge | `competitor-tools` | |
| `knowledge` | Merge | `memory` | |
| `knowledge-graph` | Merge | `memory` | |
| `latency` | Drop | — | Performance aspect, covered by article |
| `lifecycle` | Drop | — | Too generic |
| `linear` | Merge | `third-party` | |
| `long-horizon` | Merge | `context-management` | |
| `LongCLI-Bench` | Merge | `testing` | |
| `managed-policies` | Merge | `enterprise` | |
| `marketplace` | Merge | `plugins` | |
| `mcp` | Keep | `mcp` | |
| `memory` | Keep | `memory` | |
| `metrics` | Merge | `observability` | |
| `mobile` | Keep | `mobile` | |
| `model-lineage` | Merge | `model-lifecycle` | |
| `model-routing` | Merge | `model-selection` | |
| `model-selection` | Keep | `model-selection` | |
| `models` | Merge | `model-selection` | |
| `monitoring` | Merge | `observability` | |
| `multi-agent` | Merge | `subagents` | |
| `multi-tool` | Merge | `config-toml` | Multi-tool config management |
| `multimodal` | Drop | — | Feature aspect, too broad |
| `neo4j` | Merge | `third-party` | |
| `non-coding` | Keep | `non-coding` | |
| `non-interactive` | Drop | — | Covered by primary `ci-cd` |
| `notion` | Merge | `third-party` | |
| `observability` | Keep | `observability` | |
| `on-call-automation` | Merge | `incident-response` | |
| `open-source` | Keep | `open-source` | |
| `openai` | Drop | — | Too broad; everything uses OpenAI |
| `opentelemetry` | Merge | `observability` | |
| `opinion` | Keep as primary | Primary: `opinion` | |
| `orchestration` | Drop | — | Use primary category `orchestration` |
| `orchestrator` | Drop | — | Agent role, too specific |
| `oss-fund` | Merge | `open-source` | |
| `otel` | Merge | `observability` | |
| `pagerduty` | Merge | `third-party` | |
| `parallel-workflows` | Drop | — | Feature detail |
| `patch-generation` | Merge | `vulnerability-scanning` | |
| `performance` | Drop | — | Too generic; use specific tags |
| `personality` | Drop | — | Feature detail |
| `philosophy` | Drop | — | Use primary category `opinion` |
| `planner` | Drop | — | Agent role, too specific |
| `planning` | Keep | `planning` | |
| `platform` | Drop | — | Too generic |
| `plugin` | Merge | `plugins` | Normalise to plural |
| `plugin-distribution` | Merge | `plugins` | |
| `plugin-manifest` | Merge | `plugins` | |
| `plugins` | Keep | `plugins` | |
| `posttooluse` | Merge | `hooks` | |
| `pragmatic` | Drop | — | Personality trait |
| `pretooluse` | Merge | `hooks` | |
| `product-direction` | Keep | `product-direction` | |
| `production` | Drop | — | Too generic |
| `production-safety` | Merge | `incident-response` | |
| `productivity` | Drop | — | Too generic |
| `profiles` | Merge | `config-toml` | |
| `programmatic` | Drop | — | Use `typescript-sdk` or `python-sdk` |
| `prompting` | Keep | `prompting` | |
| `pull-requests` | Merge | `code-review` | |
| `quality-engineer` | Drop | — | Role name, too specific |
| `quality-gates` | Merge | `testing` | |
| `quota` | Merge | `cost-optimization` | |
| `ratatui` | Drop | — | Implementation detail |
| `rbac` | Merge | `enterprise` | |
| `real-time-coding` | Merge | `codex-spark` | |
| `remote-deployment` | Merge | `app-server` | |
| `remote-sessions` | Merge | `codex-cloud` | |
| `repository-automation` | Merge | `github-actions` | |
| `research` | Drop | — | Too generic |
| `resume` | Drop | — | CLI subcommand detail |
| `review` | Merge | `code-review` | |
| `reviewer` | Drop | — | Agent role |
| `roles` | Merge | `team-workflow` | |
| `ruler` | Drop | — | Third-party tool name |
| `rules-engine` | Merge | `approval-modes` | |
| `rust-rewrite` | Merge | `codex-rs` | |
| `sandbox` | Keep | `sandbox` | |
| `sandboxing` | Merge | `sandbox` | Normalise |
| `scheduling` | Merge | `automation` | |
| `sdk` | Drop | — | Use `typescript-sdk` or `python-sdk` |
| `security` | Drop | — | Use primary category or specific security tags |
| `session-archiving` | Drop | — | Feature detail |
| `shell-snapshot` | Drop | — | Feature detail |
| `sidebar-navigation` | Drop | — | UI detail |
| `skills` | Keep | `skills` | |
| `slack` | Merge | `third-party` | |
| `spec-driven-development` | Merge | `planning` | |
| `spokenly-mcp` | Drop | — | Specific MCP server |
| `starlark` | Merge | `approval-modes` | |
| `strategy` | Drop | — | Too generic; use primary `competitive-landscape` or `opinion` |
| `subagents` | Keep | `subagents` | |
| `swift` | Merge | `mobile` | |
| `symphony` | Merge | `team-workflow` | Symphony is a team orchestration pattern |
| `tdd` | Merge | `testing` | |
| `testing` | Keep | `testing` | |
| `thsottiaux` | Drop | — | Person name, not a tag |
| `thread-search` | Drop | — | Feature detail |
| `threads` | Drop | — | Feature detail |
| `threat-modelling` | Merge | `vulnerability-scanning` | |
| `three-person-rule` | Drop | — | Concept detail |
| `token-optimization` | Merge | `cost-optimization` | |
| `tool-comparison` | Drop | — | Use primary category `competitive-landscape` |
| `toolkit` | Merge | `non-coding` | |
| `toolkit-expansion` | Merge | `non-coding` | |
| `tracing` | Merge | `observability` | |
| `transparency` | Drop | — | Conceptual |
| `troubleshooting` | Drop | — | Use primary category or article content |
| `tui-theme` | Drop | — | Feature detail |
| `typescript` | Merge | `typescript-sdk` | |
| `undo-command` | Drop | — | Feature detail |
| `unified-exec` | Drop | — | Feature detail |
| `v0.117.0` | Drop | — | Version numbers are temporal |
| `validator` | Drop | — | Agent role |
| `value-engineer` | Drop | — | Role name |
| `view-image` | Drop | — | Feature detail |
| `visual-workflows` | Merge | `non-coding` | |
| `voice-input` | Keep | `voice-input` | |
| `voice-transcription` | Merge | `voice-input` | |
| `vs-code-sync` | Merge | `ide-extension` | |
| `vsync` | Drop | — | Third-party tool |
| `vulnerability-scanning` | Keep | `vulnerability-scanning` | |
| `wafer-scale-engine` | Drop | — | Hardware detail |
| `web-search` | Merge | `mcp` | Web search is an MCP integration |
| `websocket` | Merge | `app-server` | |
| `whisper` | Merge | `voice-input` | |
| `windows` | Keep | `windows` | |
| `wire-protocol` | Merge | `codex-rs` | |
| `workflow` | Drop | — | Too generic; use primary category |
| `workflow-design` | Drop | — | Too generic |
| `wsl` | Merge | `windows` | |
| `xcode` | Merge | `mobile` | |
| `zero-dependency` | Drop | — | Implementation detail |

---

## 4. Recommended Tag Assignment for Every Article

Format: `article-slug` -> Primary: `category` | Tags: `tag1`, `tag2`, ...

### 2026-03-26

| Slug | Primary | Secondary Tags |
|---|---|---|
| `agentic-pod-roles-and-codex` | `orchestration` | `team-workflow`, `agents-md`, `planning` |
| `agentic-primitives-codex-claude-gemini` | `competitive-landscape` | `claude-code`, `competitor-tools`, `agents-md`, `mcp` |
| `agents-md-advanced-patterns` | `configuration` | `agents-md`, `config-toml` |
| `claude-code-codex-bidirectional-mcp` | `ecosystem` | `mcp`, `claude-code` |
| `codex-cli-agents-sdk-integration` | `ecosystem` | `agents-sdk`, `mcp`, `subagents` |
| `codex-cli-approval-modes-sandbox-security` | `security` | `approval-modes`, `sandbox` |
| `codex-cli-benchmarks-real-world` | `opinion` | `testing`, `cost-optimization` |
| `codex-cli-cicd-non-interactive` | `ci-cd` | `github-actions`, `automation` |
| `codex-cli-enterprise-deployment` | `configuration` | `enterprise`, `config-toml` |
| `codex-cli-hooks-deep-dive` | `architecture` | `hooks` |
| `codex-cli-mcp-integration` | `ecosystem` | `mcp`, `plugins` |
| `codex-cli-model-selection` | `models` | `model-selection`, `cost-optimization` |
| `codex-cli-subagents-toml-parallelism` | `orchestration` | `subagents`, `config-toml` |
| `codex-cli-vs-competing-agentic-tools` | `competitive-landscape` | `claude-code`, `cursor`, `competitor-tools` |
| `codex-cli-web-search-knowledge-agents` | `ecosystem` | `mcp`, `memory` |
| `codex-cli-worktree-parallel-development` | `workflow-patterns` | `git-integration`, `automation` |
| `codex-vs-claude-code-community-opinions` | `competitive-landscape` | `claude-code`, `community-sentiment` |
| `effective-prompting-strategies` | `workflow-patterns` | `prompting`, `agents-md`, `skills` |
| `harness-engineering-symphony` | `orchestration` | `team-workflow`, `planning` |
| `migrating-claude-code-to-codex-cli` | `competitive-landscape` | `claude-code`, `config-toml` |
| `skills-ecosystem-deep-dive` | `ecosystem` | `skills`, `plugins` |
| `writing-effective-skillmd-files` | `ecosystem` | `skills`, `agents-md` |

### 2026-03-27

| Slug | Primary | Secondary Tags |
|---|---|---|
| `agents-md-bloat-problem` | `workflow-patterns` | `agents-md`, `context-management` |
| `claude-code-codex-mcp-in-practice` | `ecosystem` | `mcp`, `claude-code` |
| `codex-cli-automations-scheduled-tasks` | `ci-cd` | `automation`, `codex-cloud` |
| `codex-cli-beyond-code-file-automation` | `workflow-patterns` | `non-coding`, `automation` |
| `codex-cli-code-review-pr-integration` | `workflow-patterns` | `code-review`, `github-actions` |
| `codex-cli-figma-mcp-design-to-code` | `ecosystem` | `mcp`, `third-party` |
| `codex-cli-frontend-react-typescript` | `language-guide` | `skills`, `agents-md` |
| `codex-cli-in-2026-whats-new` | `opinion` | `product-direction`, `subagents`, `hooks`, `enterprise` |
| `codex-cli-plugins-deep-dive` | `ecosystem` | `plugins`, `skills`, `mcp` |
| `codex-cli-python-teams` | `language-guide` | `agents-md`, `automation` |
| `codex-cli-skills-ecosystem` | `ecosystem` | `skills`, `plugins`, `open-source` |
| `codex-cloud-vs-local-when-to-run-in-cloud` | `cloud` | `codex-cloud`, `cost-optimization` |
| `codex-enterprise-admin-guide` | `reference` | `enterprise`, `compliance`, `config-toml` |
| `codex-interfaces-desktop-cli-ide` | `getting-started` | `desktop-app`, `ide-extension` |
| `codex-slack-linear-cloud-tasks` | `ecosystem` | `third-party`, `codex-cloud`, `automation` |
| `compound-engineering-codex-80-20` | `orchestration` | `team-workflow`, `planning`, `code-review` |
| `context-window-management-subagent-delegation` | `architecture` | `context-management`, `subagents` |
| `multi-agent-orchestration-patterns` | `orchestration` | `subagents`, `planning` |
| `personality-difference-claude-code-vs-codex` | `competitive-landscape` | `claude-code`, `community-sentiment` |
| `planning-mode-in-practice` | `workflow-patterns` | `planning`, `config-toml` |
| `proof-of-work-principle-agents` | `opinion` | `team-workflow`, `code-review` |
| `reasoning-effort-tuning` | `models` | `model-selection`, `cost-optimization`, `config-toml` |
| `security-hardening-codex-cli` | `security` | `sandbox`, `network-security`, `approval-modes` |
| `skills-progressive-disclosure-vs-mcp` | `architecture` | `skills`, `mcp` |
| `staying-engaged-codebase-agentic-world` | `opinion` | `team-workflow`, `code-review` |
| `using-claude-code-and-codex-together` | `competitive-landscape` | `claude-code`, `mcp` |
| `what-is-codex-cli` | `getting-started` | `product-direction`, `competitor-tools` |
| `workflow-md-version-controlling-agent-behaviour` | `orchestration` | `team-workflow`, `agents-md` |

### 2026-03-28

| Slug | Primary | Secondary Tags |
|---|---|---|
| `agentic-coding-landscape-2026-antigravity-kiro` | `competitive-landscape` | `competitor-tools`, `claude-code`, `cursor` |
| `agentic-pod-in-practice-multi-agent-roles` | `orchestration` | `subagents`, `team-workflow` |
| `cc-pocket-mobile-codex-sessions` | `cloud` | `mobile`, `codex-cloud`, `approval-modes` |
| `codex-agent-loop-deep-dive` | `architecture` | `agent-loop`, `context-management` |
| `codex-app-server-json-rpc-protocol` | `architecture` | `app-server`, `typescript-sdk` |
| `codex-app-server-remote-deployment` | `architecture` | `app-server`, `credential-management` |
| `codex-app-thread-search-session-archiving` | `reference` | `desktop-app`, `ide-extension` |
| `codex-background-terminal-dev-server-workflows` | `workflow-patterns` | `automation` |
| `codex-cli-building-distributing-plugins` | `ecosystem` | `plugins`, `mcp`, `skills`, `open-source` |
| `codex-cli-commit-attribution` | `configuration` | `git-integration`, `enterprise`, `compliance` |
| `codex-cli-cost-management-token-strategy` | `models` | `cost-optimization`, `model-selection`, `enterprise`, `config-toml` |
| `codex-cli-credit-economy` | `models` | `cost-optimization` |
| `codex-cli-dbt-data-engineering` | `language-guide` | `third-party`, `agents-md` |
| `codex-cli-feature-flags-tui-tuning` | `configuration` | `config-toml` |
| `codex-cli-image-workflows` | `workflow-patterns` | `non-coding` |
| `codex-cli-mobile-xcode-ios-android` | `language-guide` | `mobile`, `mcp` |
| `codex-cli-opentelemetry-observability` | `configuration` | `observability`, `enterprise`, `config-toml` |
| `codex-cli-personality-system` | `configuration` | `agents-md`, `config-toml` |
| `codex-cli-regulated-environments-hipaa-soc2` | `security` | `compliance`, `enterprise`, `sandbox` |
| `codex-cli-thread-management-fork-resume` | `workflow-patterns` | `context-management` |
| `codex-cli-toolkit-expansion-non-code-tasks` | `workflow-patterns` | `non-coding`, `third-party`, `plugins` |
| `codex-cli-voice-input-dictation` | `ecosystem` | `voice-input`, `mcp` |
| `codex-github-copilot-coding-agent-issue-assignment` | `ecosystem` | `github-copilot`, `github-actions`, `enterprise` |
| `codex-model-lineage-context-compaction` | `architecture` | `model-lifecycle`, `context-management` |
| `codex-on-windows-native-sandbox-wsl` | `getting-started` | `windows`, `sandbox` |
| `codex-rs-rust-rewrite-architecture` | `architecture` | `codex-rs`, `sandbox` |
| `codex-security-vulnerability-discovery` | `security` | `vulnerability-scanning` |
| `codex-spark-cerebras-ultrafast-model` | `models` | `codex-spark`, `model-selection`, `config-toml` |
| `codex-subagent-path-addressing-multi-agent-v2` | `architecture` | `subagents` |
| `codex-typescript-sdk-embedding-agents` | `ecosystem` | `typescript-sdk`, `agents-sdk` |
| `cross-model-adversarial-review` | `workflow-patterns` | `code-review`, `testing` |
| `debugging-codex-agent-failures` | `reference` | `context-management`, `observability` |
| `evaluating-codex-agents-evals-longhorizon` | `workflow-patterns` | `testing` |
| `github-agentic-workflows-codex-integration` | `ci-cd` | `github-actions`, `automation` |
| `migrating-cursor-to-codex-cli` | `competitive-landscape` | `cursor`, `config-toml` |
| `migrating-github-copilot-to-codex-cli` | `competitive-landscape` | `github-copilot`, `config-toml` |
| `spec-driven-development-codex` | `workflow-patterns` | `planning`, `agents-md` |
| `test-first-development-codex-tdd-feedback-loop` | `workflow-patterns` | `testing`, `agents-md` |

### 2026-03-29

| Slug | Primary | Secondary Tags |
|---|---|---|
| `agents-md-monorepo-patterns` | `configuration` | `agents-md` |
| `codex-cli-in-2027-reading-the-roadmap` | `opinion` | `product-direction` |
| `codex-cli-incident-response-on-call-automation` | `ecosystem` | `incident-response`, `third-party`, `mcp` |
| `codex-cli-jupyter-notebooks-scientific-python` | `language-guide` | `agents-md` |
| `codex-cli-mid-turn-steering` | `workflow-patterns` | `approval-modes`, `planning` |
| `codex-cli-multi-agent-production-patterns` | `orchestration` | `subagents`, `team-workflow` |
| `codex-github-actions-complete-integration-guide` | `ci-cd` | `github-actions`, `automation` |
| `codex-max-long-horizon-tasks` | `models` | `context-management`, `model-lifecycle` |
| `codex-spark-workflow-design` | `workflow-patterns` | `codex-spark`, `model-selection`, `context-management` |
| `openai-superapp-codex-strategic-pivot` | `opinion` | `product-direction`, `codex-cloud` |
| `subagent-gotchas-known-issues` | `reference` | `subagents` |
| `vibe-coding-vs-agentic-engineering` | `opinion` | `team-workflow`, `planning` |

### 2026-03-30

| Slug | Primary | Secondary Tags |
|---|---|---|
| `codex-app-server-tui-architecture-shift` | `architecture` | `app-server`, `codex-rs` |
| `codex-app-theming-customisation` | `configuration` | `desktop-app`, `config-toml` |
| `codex-cli-approval-loop-fatigue-starlark-general-rule` | `security` | `approval-modes`, `subagents`, `sandbox` |
| `codex-cli-as-mcp-server` | `ecosystem` | `mcp`, `agents-sdk` |
| `codex-cli-config-management-multi-tool` | `configuration` | `config-toml`, `agents-md` |
| `codex-cli-docker-containerised-environments` | `configuration` | `docker`, `sandbox` |
| `codex-cli-go-teams` | `language-guide` | `agents-md`, `skills` |
| `codex-cli-hooks-engine` | `architecture` | `hooks`, `automation` |
| `codex-cli-infrastructure-as-code-terraform-pulumi-ansible` | `language-guide` | `agents-md`, `automation`, `third-party` |
| `codex-cli-java-spring-boot-teams` | `language-guide` | `agents-md`, `sandbox` |
| `codex-cli-multi-agent-v2-path-addressing-structured-messaging` | `architecture` | `subagents`, `app-server` |
| `codex-cli-neo4j-use-cases-best-practices` | `ecosystem` | `third-party`, `memory` |
| `codex-cli-network-security-domain-allowlists` | `security` | `network-security`, `credential-management`, `sandbox` |
| `codex-cli-plugin-system` | `ecosystem` | `plugins`, `skills`, `mcp` |
| `codex-cli-review-command-code-review-workflows` | `workflow-patterns` | `code-review`, `mcp` |
| `codex-cli-rules-engine-starlark-smart-approvals` | `security` | `approval-modes`, `subagents` |
| `codex-cli-session-analytics-jsonl-rollout` | `architecture` | `observability` |
| `codex-cli-thread-search-session-management` | `reference` | `context-management` |
| `codex-cli-vercel-ai-gateway-skills-plugin` | `ecosystem` | `third-party`, `plugins`, `skills` |
| `codex-python-sdk-embedding-agents` | `ecosystem` | `python-sdk` |
| `codex-spark-cerebras-real-time-coding` | `models` | `codex-spark` |
| `codex-toolkit-expansion-beyond-coding` | `opinion` | `non-coding`, `product-direction` |
| `gpt-5-codex-new-flagship-model-guide` | `models` | `model-selection`, `model-lifecycle` |
| `gpt54-mini-codex-subagent-delegation` | `models` | `model-selection`, `subagents`, `cost-optimization` |
| `graphiti-agent-memory-store` | `ecosystem` | `memory`, `third-party` |
| `gstack-garry-tan-production-skills-toolkit` | `ecosystem` | `skills`, `claude-code` |
| `reddit-codex-cli-sentiment-march-2026` | `opinion` | `community-sentiment` |

### 2026-03-31

| Slug | Primary | Secondary Tags |
|---|---|---|
| `codex-cli-app-server-remote-websocket` | `architecture` | `app-server`, `codex-cloud` |
| `codex-cli-apply-patch-v4a-diff-format` | `architecture` | `agent-loop` |
| `codex-cli-context-compaction-architecture` | `architecture` | `context-management`, `config-toml` |
| `codex-cli-custom-model-providers` | `configuration` | `model-selection`, `credential-management`, `config-toml` |
| `codex-cli-dotnet-csharp-teams` | `language-guide` | `agents-md`, `sandbox`, `third-party` |
| `codex-cli-network-security-requirements-toml` | `security` | `network-security`, `sandbox` |
| `codex-cli-pretooluse-posttooluse-hooks` | `ci-cd` | `hooks`, `testing`, `automation` |
| `codex-cli-profiles-configuration-switching` | `configuration` | `config-toml` |
| `codex-cli-python-sdk-app-server-rpc` | `ecosystem` | `python-sdk`, `app-server` |
| `codex-cli-realtime-sessions-voice-transcription` | `ecosystem` | `voice-input`, `config-toml` |
| `codex-cli-rust-teams` | `language-guide` | `agents-md`, `codex-rs` |
| `codex-open-source-fund` | `opinion` | `open-source`, `community-sentiment` |
| `codex-plugin-cc-cross-model-bridge` | `ecosystem` | `plugins`, `claude-code` |
| `codex-security-agent-vulnerability-scanning-threat-modelling` | `security` | `vulnerability-scanning`, `automation` |
| `codex-spark-cerebras-real-time-coding` | `models` | `codex-spark` |
| `gpt54-computer-use-tool-search-codex-cli` | `models` | `model-selection`, `plugins` |

### 2026-04-01

| Slug | Primary | Secondary Tags |
|---|---|---|
| `ai-slopageddon-open-source-contribution-crisis` | `opinion` | `open-source`, `community-sentiment` |
| `claude-code-open-source-architecture` | `competitive-landscape` | `claude-code`, `agent-loop` |
| `claude-code-source-leak-what-it-reveals` | `competitive-landscape` | `claude-code`, `agent-loop` |
| `codex-cli-authentication-flows-credential-management` | `security` | `credential-management`, `config-toml` |
| `codex-cli-kubernetes-cloud-native-teams` | `language-guide` | `kubernetes`, `agents-md`, `sandbox` |
| `codex-cli-multi-agent-cookbook-pattern` | `orchestration` | `agents-sdk`, `subagents` |
| `codex-cli-triggers-event-driven-github-automation` | `ci-cd` | `github-actions`, `automation` |
| `codex-cli-windows-native-sandbox-wsl` | `getting-started` | `windows`, `sandbox` |
| `codex-cloud-exec-best-of-n-attempts` | `cloud` | `codex-cloud`, `testing` |
| `codex-exec-unix-pipeline-structured-output` | `ci-cd` | `automation` |
| `codex-ide-extension-vs-code-jetbrains` | `cloud` | `ide-extension`, `desktop-app`, `codex-cloud` |
| `codex-plugins-skills-mcp-authoring` | `ecosystem` | `plugins`, `skills`, `mcp` |
| `codex-responses-api-phase-parameter-custom-harness` | `architecture` | `agent-loop`, `agents-sdk` |

### 2026-04-06

| Slug | Primary | Secondary Tags |
|---|---|---|
| `codex-cli-persistent-memory-mcp-servers` | `ecosystem` | `memory`, `mcp`, `context-management` |

### 2026-04-07

| Slug | Primary | Secondary Tags |
|---|---|---|
| `agents-md-open-standard-cross-tool-portability` | `opinion` | `agents-md`, `open-source` |
| `cli-agent-saas-readiness-matrix` | `competitive-landscape` | `competitor-tools`, `enterprise` |
| `codex-cli-agentic-loop-internals` | `architecture` | `agent-loop`, `codex-rs` |
| `codex-cli-competitive-position-april-2026` | `competitive-landscape` | `claude-code`, `community-sentiment`, `product-direction` |
| `codex-cli-diagnostic-toolkit-tracing-sandbox-testing` | `reference` | `observability`, `sandbox` |
| `codex-cli-forward-deployed-engineer` | `workflow-patterns` | `team-workflow`, `prompting` |
| `codex-cli-gitlab-integration-duo-agent-platform` | `ci-cd` | `gitlab`, `mcp` |
| `codex-cli-model-lifecycle-deprecations-migrations` | `models` | `model-lifecycle`, `config-toml` |
| `codified-context-three-tier-knowledge-architecture` | `architecture` | `agents-md`, `skills`, `context-management` |
| `cross-model-review-loop-automation` | `workflow-patterns` | `code-review`, `skills`, `automation` |
| `embedding-ai-agents-saas-codex-opencode-pi` | `competitive-landscape` | `competitor-tools`, `typescript-sdk` |
| `goose-mcp-native-architecture` | `competitive-landscape` | `competitor-tools`, `mcp` |
| `learning-plan-becoming-codex-cli-expert` | `getting-started` | `agents-md`, `planning` |
| `sourcegraph-amp-multi-repo-orchestrator` | `competitive-landscape` | `competitor-tools`, `subagents` |

### 2026-04-08

| Slug | Primary | Secondary Tags |
|---|---|---|
| `bootstrapping-agents-md` | `getting-started` | `agents-md`, `config-toml` |
| `cline-cli-yolo-mode-automation` | `competitive-landscape` | `competitor-tools`, `automation` |
| `codex-cli-approval-framework` | `security` | `approval-modes`, `config-toml` |
| `codex-cli-configuration-reference` | `reference` | `config-toml` |
| `codex-cli-javascript-repl-stateful-scripting` | `workflow-patterns` | `automation` |
| `codex-cli-memory-internals` | `architecture` | `memory`, `context-management` |
| `codex-cli-token-costs-billing` | `models` | `cost-optimization` |
| `codex-cli-tui-shortcuts-slash-commands` | `reference` | `config-toml` |
| `codex-cloud-task-application` | `cloud` | `codex-cloud`, `third-party`, `automation` |
| `codex-desktop-automations` | `cloud` | `desktop-app`, `automation` |
| `codex-exec-jsonl-reference` | `reference` | `automation`, `observability` |
| `codex-github-action` | `ci-cd` | `github-actions`, `automation` |
| `codex-plugin-discovery` | `ecosystem` | `plugins` |
| `codex-sandbox-platform-implementation` | `architecture` | `sandbox`, `windows` |
| `codex-typescript-sdk-streaming-multimodal` | `ecosystem` | `typescript-sdk` |
| `cross-platform-agent-portability-skill-md` | `ecosystem` | `skills`, `agents-md` |
| `cross-surface-session-sync` | `cloud` | `codex-cloud`, `desktop-app`, `ide-extension` |
| `four-surface-architecture` | `architecture` | `desktop-app`, `ide-extension`, `codex-cloud` |
| `installing-codex-cli` | `getting-started` | `config-toml` |
| `plan-mode-mechanics` | `workflow-patterns` | `planning` |
| `tessl-skill-evaluation-framework` | `ecosystem` | `skills`, `testing` |

---

## Summary Statistics

- **Total articles:** 191
- **Primary categories used:** 13
- **Approved secondary tags:** 47
- **Current unique tags in use:** ~230 (many one-off)
- **Tags dropped:** ~100 (too specific, too generic, redundant, or temporal)
- **Tags merged:** ~80 (synonyms, near-duplicates, product-specific names rolled into broader tags)
- **Tags kept as-is:** ~30

### Primary Category Distribution (approximate)

| Category | Count |
|---|---|
| `ecosystem` | 30 |
| `architecture` | 22 |
| `competitive-landscape` | 18 |
| `workflow-patterns` | 21 |
| `models` | 14 |
| `security` | 14 |
| `configuration` | 16 |
| `ci-cd` | 10 |
| `opinion` | 14 |
| `cloud` | 8 |
| `language-guide` | 13 |
| `reference` | 8 |
| `getting-started` | 6 |
| `orchestration` | 9 |
