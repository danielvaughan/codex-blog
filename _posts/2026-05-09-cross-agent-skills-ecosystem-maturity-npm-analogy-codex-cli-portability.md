---
title: "Cross-Agent Skills Hit the npm Moment: 351K Skills, Three Marketplaces, and a Portability Standard"
description: "A Termdock analysis published in May 2026 makes a compelling case: the Agent Skills ecosystem is reaching its npm circa 2011 inflection point."
date: 2026-05-09T00:00:00+00:00
last_modified_at: 2026-08-16T14:12:29+01:00
category: ecosystem
tags: [skills, SKILL.md, cross-agent, npm, portability, marketplace, SkillsMP, Skills.sh, ClawHub, agent-skills-cli]
source: https://www.termdock.com/blog/cross-agent-skills-new-npm
related:
  - skills-marketplace-landscape-may-2026
  - codex-cli-plugin-marketplace-remote-install-workspace-sharing-bundled-hooks
parent: "Articles"
nav_order: 584
type: Technical Article
timestamp: 2026-05-09T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-05-09-cross-agent-skills-ecosystem-maturity-npm-analogy-codex-cli-portability"
---
![Sketchnote diagram for: Cross-Agent Skills Hit the npm Moment: 351K Skills, Three Marketplaces, and a Portability Standard](/sketchnotes/articles/2026-05-09-cross-agent-skills-ecosystem-maturity-npm-analogy-codex-cli-portability.png)


# Cross-Agent Skills Hit the npm Moment

A Termdock analysis published in May 2026 makes a compelling case: the Agent Skills ecosystem is reaching its "npm circa 2011" inflection point — a standardised format (SKILL.md), fragmented distribution, incomplete tooling, but foundational conventions that are solidifying fast.

## The Numbers

| Marketplace | Skills Listed | Security Scanning | Key Feature |
|------------|--------------|-------------------|-------------|
| SkillsMP   | 351K+        | No                | Semantic search, REST API, 9-language UI |
| Skills.sh  | 83K          | Yes (Snyk)        | Vercel-backed, telemetry-based popularity |
| ClawHub    | ~3,200       | Manual curation   | Curated quality over quantity |

Total ecosystem: **351,000+ published skills** across three competing marketplaces. A 13.4% critical vulnerability rate in scanned skills remains a significant concern — security tooling is the clearest gap versus npm's mature audit pipeline.

## The Portability Promise

Six major agents now support SKILL.md, but each expects a different directory:

| Agent | Skills Directory |
|-------|-----------------|
| Claude Code | `.claude/skills/` |
| Codex CLI | `.agents/skills/` (primary) |
| GitHub Copilot | `.github/skills/` |
| Gemini CLI | `.gemini/skills/` |
| Cursor | `.cursor/skills/` |
| Windsurf | `.windsurf/skills/` |

The emerging convention: `.agents/skills/` as a universal fallback. Codex CLI already uses this as its primary directory, which positions it well for cross-agent portability.

## agent-skills-cli: The Universal Package Manager

The [agent-skills-cli](https://github.com/Karanjot786/agent-skills-cli) tool (`npm install -g agent-skills-cli`) acts as a universal skills package manager across 45 agent platforms. Key commands:

- `skills install @owner/skill-name` — install to all detected agents
- `skills search [query]` — interactive FZF-powered discovery
- `skills compose` — compose multiple skills
- `skills sandbox` — test skills in isolation
- `-a codex` — target a specific agent

This is the closest equivalent to npm's cross-project install experience for agent skills.

## Cross-Agent Authoring Rules

Four principles for skills that work everywhere:

1. **Basic frontmatter only** — name and description; avoid agent-specific metadata
2. **Outcome-oriented language** — describe what should happen, not which tools to call
3. **Relative paths** — never assume a specific workspace layout
4. **Test on two+ agents** — portability is only real when verified

## Why This Matters for Enterprise / Agentic Pod

For Daniel's agentic pod work, the implications are:

- **Skill portability** means the same governance skills (security review, code quality, deployment checklists) work whether the team uses Codex CLI, Claude Code, or Cursor
- **Registry consolidation** is likely — Skills.sh (Vercel + Snyk) looks positioned to become the npm equivalent
- **Security scanning gaps** mean enterprise teams must run their own skill audit pipeline until marketplace scanning matures
- **skillpm** maps skills onto npm's semver and dependency system — worth watching for enterprise version pinning

## Prediction

The article forecasts three convergences:
1. Skills.sh emerges as the dominant registry (Vercel backing + security scanning)
2. Skills become npm packages with metadata, not a separate distribution channel
3. Agents adopt `.agents/skills/` as primary directory instead of fallback

This mirrors npm's own consolidation arc — fragmented registries, then a single winner, then tight build-tool integration.
