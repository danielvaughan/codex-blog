---
title: "Codex's Three-Tier Browser Architecture: Plugins, Chrome Extension, and Sandboxed Browser"
description: "Codex implements a hierarchical browser tool system that solves a fundamental challenge in agentic systems: tool selection as a first-class problem. Rather."
date: 2026-05-17T00:00:00+00:00
last_modified_at: 2026-08-07T18:18:49+01:00
category: architecture
tags: [browser-use, chrome-extension, plugins, tool-selection, sandboxing, agentic-patterns]
source: https://www.verdent.ai/guides/codex-three-tier-browser-architecture
parent: "Articles"
nav_order: 936
type: Technical Article
timestamp: 2026-05-17T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-05-17-codex-three-tier-browser-architecture-plugins-chrome-extension-sandboxed-tool-selection"
---
![Sketchnote diagram for: Codex's Three-Tier Browser Architecture: Plugins, Chrome Extension, and Sandboxed Browser](/sketchnotes/articles/2026-05-17-codex-three-tier-browser-architecture-plugins-chrome-extension-sandboxed-tool-selection.png)

# Codex's Three-Tier Browser Architecture

Codex implements a hierarchical browser tool system that solves a fundamental challenge in agentic systems: tool selection as a first-class problem. Rather than letting models freely choose among tools, Codex hard-codes selection priorities based on trust boundaries and sandboxing requirements.

## The Three Tiers

### Tier 1: Dedicated Plugins (highest priority)

Plugin integrations provide structured API access to services (GitHub, Slack, Figma, Linear, etc.). Benefits:

- API-grade reliability and predictable schemas
- Cleaner audit trails
- Faster performance for well-defined operations
- No browser state required

### Tier 2: Chrome Extension (signed-in browser)

The Chrome extension (v1.1.4, released May 7 2026) operates inside your actual Chrome profile with existing cookies and authentication tokens. Use cases:

- Internal dashboards and staging environments
- Services requiring user sessions (LinkedIn, Salesforce, Gmail)
- Any site reachable through your current login state
- Runs in isolated Chrome tabs (doesn't commandeer active browsing)

Explicit invocation: `@Chrome open [tool] and do [thing]`

### Tier 3: In-App Browser (sandboxed)

Built on Atlas technology. Never reads from or writes to your Chrome profile. Handles:

- Localhost development servers
- Local file previews
- Public pages without authentication requirements

## Tool Selection Mechanism

Deterministic priority stack: **plugins -> Chrome -> in-app browser**

Codex analyses the task description to select the appropriate tier automatically:

1. If a dedicated plugin exists for the service, use it
2. If the task requires authenticated browser access and no plugin exists, use Chrome
3. If the task is local or doesn't require authentication, use in-app browser

Developers can override automatic selection with explicit `@Chrome` syntax.

## Implications for Agentic Workflows

**Predictability over flexibility**: By hard-coding selection priorities, Codex reduces tool selection errors that compound with task complexity. This is a deliberate architectural choice - trading model autonomy for deterministic routing.

**Trust boundaries**: Each tier operates within distinct security perimeters. Plugins have scoped API access, Chrome has user-session scope, and the in-app browser is fully sandboxed.

**Enterprise governance**: The tiered model means administrators can control which plugins are available, which sites Chrome can access (allowlist/blocklist), and what the sandboxed browser can reach - all independently.

## Key Takeaway

"Tool selection is a first-class problem in agentic systems." The three tiers are designed to work together, not replace each other. More precise tools are always preferred when available, with fallback mechanisms for less-structured interactions.
