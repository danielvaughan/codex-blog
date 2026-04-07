---
title: "Managing Codex CLI Configuration Across Multiple AI Tools"
date: 2026-03-30
tags: [config-management, multi-tool, ruler, cc-switch, vsync, caliber, agents-md]
---

*Most teams don't just use Codex CLI. They use Codex alongside Claude Code, Cursor, Gemini CLI, or GitHub Copilot. This creates a configuration management problem: AGENTS.md, config.toml, MCP servers, and skills all need to stay in sync — or deliberately diverge — across tools. This article covers the patterns and tools that solve it.*

---

## The Problem: Configuration Drift

When you use a single AI coding tool, configuration is simple: one `AGENTS.md`, one `config.toml`, one set of MCP servers. Add a second tool and the problem compounds:

- Codex uses `AGENTS.md` and `~/.codex/config.toml`
- Claude Code uses `CLAUDE.md` and `~/.config/claude/settings.json`
- Cursor uses `.cursorrules` and `.cursor/settings.json`
- Gemini CLI uses `AGENTS.md` and `~/.gemini/config.json`

Rules you write for one tool don't automatically apply to others. MCP servers configured in Codex aren't available to Claude Code. Project conventions documented in `AGENTS.md` silently diverge from `CLAUDE.md` as both files evolve independently.

This is **configuration drift** — the slow uncoupling of your tool configs from your actual intentions.

---

## Pattern 1: Single Source of Truth with Ruler

**[ruler](https://github.com/intellectronica/ruler)** (2.6K stars) solves drift by making you the authority once:

```
your-repo/
  .ruler/
    global.md       ← applies to all tools
    codex.md        ← Codex-specific overrides
    claude.md       ← Claude Code-specific overrides
  ruler.toml        ← tool routing config
```

Example `ruler.toml`:

```toml
[agents.codex]
output = "AGENTS.md"
extra_files = [".codex/config.toml"]
skills = true
mcp = true

[agents.claude]
output = "CLAUDE.md"

[agents.cursor]
output = ".cursorrules"
```

Run `ruler sync` and Ruler generates each tool's config file from the `.ruler/` source. The global rules appear in all outputs; tool-specific files add or override only where needed.

**Why this matters for Codex CLI specifically:** When you update your Codex AGENTS.md with new project conventions, those changes automatically propagate to Claude Code's CLAUDE.md on the next `ruler sync`. No more stale instructions in one tool while the other is current.

---

## Pattern 2: Provider Switching with cc-switch

**[cc-switch](https://github.com/farion1231/cc-switch)** (35K+ stars — the highest-star community tool in the Codex ecosystem) addresses a different problem: switching between API providers or account configurations.

Use cases:
- Switch between OpenAI production keys and a testing key with limited credits
- Switch a teammate's machine to the team's shared API key for a pairing session
- Route Codex to a specific Azure OpenAI endpoint for enterprise compliance

cc-switch manages `~/.codex/config.toml` automatically, editing the `[model_providers]` and `api_key` fields without you touching the file directly. It also handles Claude Code's separate config format and Gemini CLI simultaneously — useful if you're testing the same task against multiple model providers.

**Config template pattern:** Store a set of named config profiles in cc-switch, then switch with one click (desktop app) or one command:

```bash
cc-switch use openai-production    # → sets ~/.codex/config.toml
cc-switch use azure-compliance     # → sets Azure endpoint + credentials
cc-switch use local-ollama         # → sets local model endpoint
```

---

## Pattern 3: MCP Config Sync with vsync

**[vsync](https://github.com/nicepkg/vsync)** solves MCP server configuration sync. When you add a new MCP server to Codex (`~/.codex/config.toml`), vsync can propagate that configuration to Claude Code's JSON settings and Cursor's JSONC config automatically.

```bash
vsync pull    # reads all configured tool configs
vsync push    # writes MCP server list to all tools
```

vsync handles format conversion automatically: Codex uses TOML, Claude Code uses JSON, Cursor uses JSONC. A Figma MCP server configured once in vsync appears correctly formatted in all three tool configs.

**Limitation:** vsync syncs MCP configs but not skills or AGENTS.md content — for those, use Ruler.

---

## Pattern 4: Codebase-Aware Config with Caliber

**[caliber](https://github.com/caliber-ai-org/ai-setup)** takes a different approach: instead of syncing a single source of truth across tools, it *regenerates* configs from your codebase.

```bash
caliber init         # fingerprints repo → generates AGENTS.md + config.toml
caliber score        # evaluates config quality (no LLM calls)
caliber update       # re-fingerprints after major code changes
```

When your codebase shifts — a new language added, a new framework adopted, a security library integrated — `caliber update` detects the drift and proposes config updates. It also auto-configures relevant MCP servers based on detected frameworks (Figma MCP for design-heavy projects, Datadog MCP for observability setups, etc.).

**Best use case:** Initial project setup and periodic config health checks (`caliber score` in CI). Less useful for day-to-day config maintenance where Ruler's explicit source-of-truth model is more predictable.

---

## Quick Decision Guide

| Scenario | Best Tool |
|----------|-----------|
| You use Codex + Claude Code and want instructions in sync | **Ruler** |
| You switch API providers/accounts frequently | **cc-switch** |
| You want MCP servers to appear in all your AI tools | **vsync** |
| You want configs auto-generated from your codebase | **caliber** |
| New project setup, not sure what config to write | **caliber** then **ruler** |
| Enterprise: lock all team machines to approved providers | **cc-switch** with MDM |

---

## The Layered Stack

These tools compose well:

1. **caliber** bootstraps a high-quality starting config from your codebase
2. **ruler** becomes the ongoing source of truth for rule content
3. **vsync** ensures MCP servers are consistent across tools
4. **cc-switch** handles provider/credential switching day-to-day

For solo developers, Ruler alone is often enough. For teams using 3+ AI coding tools on shared codebases, the full stack prevents the slow config entropy that accumulates when each tool's config is manually maintained.

---

## Codex-Specific Considerations

When using any of these tools with Codex CLI, be aware:

- **`AGENTS.md` is hierarchical**: Ruler should write global rules to the repo-root `AGENTS.md` and service-specific rules to service-level files. Don't flatten everything into one file.
- **`config.toml` has two layers**: `~/.codex/config.toml` (user-global) and `.codex/config.toml` (project-local). cc-switch and vsync typically target the user-global layer; project-level configs should be committed to the repo and managed manually or via caliber.
- **Skills and plugins are Codex-specific**: Neither ruler nor vsync currently syncs skills — these must be managed per-tool. If you use the same skill in Codex and Claude Code, install it separately in each.

*Published: 2026-03-30*
