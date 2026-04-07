---
title: "Codex CLI Skills Ecosystem"
---

![Sketchnote diagram for: Codex CLI Skills Ecosystem](/sketchnotes/articles/2026-03-26-skills-ecosystem-deep-dive.png)

*Researched 2026-03-26. Focus: where to find and install skills for Codex CLI.*

---

## What Are Skills?

Skills are self-contained directories with a `SKILL.md` file (plus optional scripts and references) that extend Codex with domain-specific knowledge and workflows.[^1] Think of them as "onboarding guides" for a task area — packaging procedural knowledge so Codex follows a workflow reliably and repeatably.

Codex uses **progressive disclosure**: it loads only skill metadata (name, description) at session start, then loads the full `SKILL.md` instructions on demand when a task matches.[^2] This keeps context efficient.

**Key difference from MCP:** MCP operates at the tool/transport layer (gives Codex access to external tools). Skills operate at the instruction layer (tell Codex *how* to do something). The two are complementary — skills can declare MCP dependencies in `agents/openai.yaml`.[^3]

---

## Skill File Structure

```
my-skill/
  SKILL.md              # Required: name, description, instructions
  agents/openai.yaml    # UI metadata for skill lists
  scripts/              # Python/Bash scripts
  references/           # Docs loaded into context on demand
  assets/               # Templates, icons, etc.
```

**`SKILL.md` minimum format:**[^4]

```yaml
---
name: my-skill
description: What this skill does (used for implicit matching)
---
# Instructions for Codex...
```

---

## Skill Load Scopes (Priority Order)

1. **Repository**: `.agents/skills/` in your project (committed to repo)[^5]
2. **User**: `$HOME/.agents/skills/` (personal skills)[^5]
3. **Admin**: `/etc/codex/skills/` (org-managed)[^5]
4. **System**: Built-in skills bundled with Codex[^5]

---

## Installing Skills

**Via `$skill-installer` inside Codex:**[^6]

```
$skill-installer gh-fix-ci       # install curated skill by name
$skill-installer gh-address-comments
```

**Via `npx skills` CLI:**[^7]

```bash
npx skills add vercel-labs/agent-skills   # install from GitHub
npx skills find "react testing"           # search
npx skills update                         # update all installed
```

**Via plugin marketplace (Claude Code compatible):**

```
/plugin marketplace add ComposioHQ/awesome-codex-skills
```

After installing, restart Codex to detect new skills.[^6] To disable without deleting, use `~/.codex/config.toml`. ⚠️ [unverified]

---

## Where to Find Skills

### 1. Official: github.com/openai/skills

- **URL:** https://github.com/openai/skills
- **Status:** ~13K stars, 726 forks, **35 curated skills** as of March 2026 ⚠️ [unverified][^8]
- **Tiers:**[^9]
  - **System** (auto-bundled): `skill-creator`, `skill-installer`
  - **Curated**: reviewed and maintained by OpenAI
  - **Experimental**: community-submitted, less vetted

**Featured curated skills:**[^10]

| Skill | What it does |
|-------|-------------|
| `gh-address-comments` | Resolve GitHub PR review feedback automatically |
| `gh-fix-ci` | Diagnose and fix failing CI builds |
| Vercel / Netlify / Cloudflare / Render | Deployment workflows for each platform |
| Figma | Implement designs from Figma files into code |
| Jupyter notebooks | Notebook-specific workflows |
| ASP.NET Core / WinUI | Framework-specific coding patterns |

**OpenAI's own use:** Between Dec 2025–Feb 2026, the Agents SDK repos merged **457 PRs** (up from 316 the prior quarter)[^11] using repo-local skills + AGENTS.md + GitHub Actions for repeatable engineering workflows.[^11]

**Docs:** https://developers.openai.com/codex/skills

---

### 2. Community: ComposioHQ/awesome-codex-skills

- **URL:** https://github.com/ComposioHQ/awesome-codex-skills
- A curated list of practical Codex skills for automating workflows[^12]
- Notable community skills: `brand-guidelines/`, `canvas-design/`, `image-enhancer/`, `slack-gif-creator/`, `theme-factory/`[^12]
- Cross-compatible with Claude Code and other agents[^12]

---

### 3. Cross-Agent: skills-directory/skill-codex

- **URL:** https://github.com/skills-directory/skill-codex
- A **Claude Code skill** that delegates prompts to Codex CLI — lets Claude Code hand off tasks to Codex[^13]
- Useful for hybrid Claude Code + Codex workflows (Daniel's "agentic pod" pattern)
- Install in Claude Code: `/plugin marketplace add skills-directory/skill-codex`[^13]

---

### 4. Marketplace: SkillsMP

- **URL:** https://skillsmp.com
- 500,000+ agent skills with filtering by category, author, popularity[^14]
- Compatible with Claude Code, Codex CLI, ChatGPT, Gemini CLI, Cursor, Aider, Windsurf, and 4 more ⚠️ [unverified][^14]
- Good for browsing the broad ecosystem

---

### 5. Simon Willison's Coverage (Dec 2025)

Simon Willison noted in December 2025 that OpenAI was "quietly adopting skills"[^15] — the format originated with Anthropic's Claude Code and OpenAI adopted it.[^15] His post: https://simonwillison.net/2025/Dec/12/openai-skills/

---

## Cross-Platform Compatibility

Skills (with `SKILL.md`) work natively across **11 tools**: Claude Code, OpenAI Codex, Gemini CLI, Cursor, Aider, Windsurf, OpenCode, Augment, Kilo Code, Antigravity, OpenClaw. ⚠️ [unverified][^16]

This means skills you write for Codex are reusable across your agentic pod (e.g., a Claude Code subagent can also use a Codex-authored skill).

---

## Creating Skills for the Agentic Pod

For Daniel's agentic pod use case, consider skills for each agent role:

```
.agents/skills/
  architect-review/     # SKILL.md: system design review checklist
  qa-coverage/          # SKILL.md: test coverage analysis workflow
  ci-fix/               # SKILL.md: diagnose and fix CI failures
  pr-standards/         # SKILL.md: company PR description format
```

Skills committed to `.agents/skills/` are shared across the whole team — when anyone runs Codex in the repo, these skills are available automatically.[^5]

---

## Key Links

- Official catalog: https://github.com/openai/skills
- Official docs: https://developers.openai.com/codex/skills
- Community skills: https://github.com/ComposioHQ/awesome-codex-skills
- SkillsMP: https://skillsmp.com
- Cross-agent delegation: https://github.com/skills-directory/skill-codex
- Simon Willison's intro: https://simonwillison.net/2025/Dec/12/openai-skills/

---

## Citations

[^1]: Agent Skills specification — what a skill is and its directory structure: https://agentskills.io/what-are-skills

[^2]: Progressive disclosure — three-tier loading (metadata at startup, full SKILL.md on activation, resources on demand): https://agentskills.io/specification#progressive-disclosure

[^3]: `agents/openai.yaml` for UI metadata and MCP dependencies in Codex skills: https://developers.openai.com/codex/skills

[^4]: SKILL.md minimum format — required `name` and `description` frontmatter fields: https://agentskills.io/specification

[^5]: Codex skill load scopes (repository `.agents/skills/`, user `$HOME/.agents/skills/`, admin `/etc/codex/skills/`, system built-in): https://developers.openai.com/codex/skills and https://vibecoding.app/blog/openai-skills-review

[^6]: Installing skills via `$skill-installer` and restarting Codex: https://developers.openai.com/codex/skills

[^7]: `npx skills` CLI for installing skills from GitHub: https://skillsmp.com/skills/doitian-skills-repo-skills-skill-installer-skill-md

[^8]: openai/skills repository — the live star/fork count as of March 2026 is ~15,400 stars and 912 forks (not ~13K/726 as stated); the 35-curated-skills figure is cited by third-party reviews but not confirmed on the repo page itself: https://github.com/openai/skills and https://vibecoding.app/blog/openai-skills-review

[^9]: Three skill tiers (system, curated, experimental) in the openai/skills catalog: https://github.com/openai/skills

[^10]: Featured curated skills including `gh-address-comments`, `gh-fix-ci`, and platform deployment skills: https://github.com/ComposioHQ/awesome-codex-skills

[^11]: Between Dec 2025–Feb 2026 the Agents SDK repos merged 457 PRs, up from 316 the prior quarter, using repo-local skills + AGENTS.md + GitHub Actions: https://developers.openai.com/blog/skills-agents-sdk

[^12]: ComposioHQ/awesome-codex-skills — curated Codex skills including brand-guidelines, canvas-design, image-enhancer, slack-gif-creator, theme-factory: https://github.com/ComposioHQ/awesome-codex-skills

[^13]: skills-directory/skill-codex — Claude Code skill that delegates prompts to Codex CLI: https://github.com/skills-directory/skill-codex

[^14]: SkillsMP marketplace — 500,000+ agent skills with category/author/popularity filtering: https://skillsmp.com and https://smartscope.blog/en/blog/skillsmp-marketplace-guide/

[^15]: Simon Willison's December 2025 coverage: "OpenAI aren't talking about it yet, but it turns out they've adopted Anthropic's brilliant 'skills' mechanism in a big way": https://simonwillison.net/2025/Dec/12/openai-skills/ (via https://fedi.simonwillison.net/@simon/115709204111716831)

[^16]: The agentskills.io homepage lists over 30 compatible tools as of March 2026 (including Claude Code, OpenAI Codex, Gemini CLI, Cursor, OpenCode, and many more), but the specific list of 11 tools named here (Aider, Windsurf, Augment, Kilo Code, Antigravity, OpenClaw) could not be fully verified against the agentskills.io compatibility list: https://agentskills.io
