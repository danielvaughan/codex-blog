---
title: "AGENTS.md Advanced Patterns: Nested Hierarchies, Override Files and Fallbacks"
layout: single
date: 2026-03-26
---

![Sketchnote: AGENTS.md Advanced Patterns: Nested Hierarchies, Override Files and Fallbacks](/sketchnotes/articles/2026-03-26-agents-md-advanced-patterns.png)


> The basic three-tier hierarchy (`~/.codex/AGENTS.md` → repo root → subdirectory) is documented everywhere. This article covers what isn't: override files, fallback filenames, size limits, monorepo patterns, and debugging the active instruction chain.

---

## The Full Discovery Order

Codex builds its instruction chain on **every run** by walking the filesystem.[^1] The full resolution order is:

```
1. ~/.codex/AGENTS.override.md    ← highest precedence global override
   OR ~/.codex/AGENTS.md          ← standard global defaults

2. <git-root>/AGENTS.override.md  ← repo override (beats repo AGENTS.md)
   OR <git-root>/AGENTS.md        ← standard repo rules

3. <intermediate-dirs>/AGENTS.override.md  ← per each dir between root and cwd
   OR <intermediate-dirs>/AGENTS.md

4. <cwd>/AGENTS.override.md       ← directory you're working in
   OR <cwd>/AGENTS.md
```

**Key rules:**

- Files concatenate from root downward — later files win on conflicts[^1]
- `AGENTS.override.md` takes precedence over `AGENTS.md` at the *same* level[^1]
- Empty files are silently skipped[^1]
- Discovery stops at `project_doc_max_bytes` (default: **32 KiB**)[^2]
- Codex rebuilds this chain fresh on every run — no caching to clear[^1]

---

## Override Files: When and Why

`AGENTS.override.md` files let you layer temporary or role-specific instructions **without modifying the shared `AGENTS.md`** that others depend on.[^1]

### Pattern 1: Temporary personal override (global)

You're debugging a payment bug and need Codex to always run the payments test suite:

```bash
# ~/.codex/AGENTS.override.md (temporary — delete when done)
## Temporary Override: Payments Debug Session
- Always run `make test-payments` after any file change in services/payments/
- Focus on the checkout flow; ignore unrelated failures
```

Delete the file when done. The base `~/.codex/AGENTS.md` is untouched.

### Pattern 2: Team subdirectory with different rules

In a monorepo, `services/payments/` needs different test commands and stricter constraints than the root:

```
my-monorepo/
├── AGENTS.md                        ← company-wide defaults
├── services/
│   ├── payments/
│   │   ├── AGENTS.override.md       ← payments-team rules (override)
│   │   └── src/
│   └── billing/
│       └── AGENTS.md                ← billing-team additions (additive)
```

```markdown
# services/payments/AGENTS.override.md
## Payments Service Rules

**Test command:** `make test-payments` (NOT the root `npm test`)
**Never modify:** `src/generated/`, `schema/migrations/`
**Security rule:** All PII fields must be annotated with `@PII` — flag any unannotated additions
**Before committing:** Run `make audit-pci` — must exit 0
```

### Pattern 3: CI/CD pipeline override

In CI, you want Codex to skip interactive prompts and apply stricter rules:

```bash
# In your GitHub Actions workflow:
cat > .codex/AGENTS.override.md << 'EOF'
## CI Mode
- Never ask clarifying questions — make your best judgment
- Always run the full test suite before completing any task
- If tests fail, explain the failure and stop — do not attempt a fix
- Do not make commits; only propose changes
EOF
codex exec "$TASK" --full-auto
rm .codex/AGENTS.override.md
```

---

## Fallback Filenames

If your team already uses a different file for developer guidance (e.g., `TEAM_GUIDE.md`, `CONTRIBUTING.md`, `.agents.md`), you can tell Codex to treat it as equivalent to `AGENTS.md`:[^3]

```toml
# ~/.codex/config.toml
[agent]
project_doc_fallback_filenames = ["TEAM_GUIDE.md", ".agents.md", "CONTRIBUTING.md"]
```

Codex checks for these in addition to `AGENTS.md` and `AGENTS.override.md` at each directory level, using the same priority order.[^1]

**Use case:** Adopting Codex on a team that already has a well-maintained `CONTRIBUTING.md` — no migration needed. Tell Codex to read it, and gradually migrate content to a dedicated `AGENTS.md` over time.

---

## Managing the 32 KiB Size Limit

Combined instruction files are capped at **32 KiB** (`project_doc_max_bytes`).[^2] Beyond this, Codex truncates silently.[^4]

### Raising the limit

```toml
# ~/.codex/config.toml
[agent]
project_doc_max_bytes = 65536   # 64 KiB
```

Raising the limit increases token usage. Prefer splitting instructions instead.[^2]

### Splitting across nested directories

Rather than one giant repo-root `AGENTS.md`, distribute rules to the directories that need them:

```
project/
├── AGENTS.md                ← ~4 KiB: repo layout, global conventions, top-level commands
├── src/
│   ├── api/
│   │   └── AGENTS.md        ← ~2 KiB: API-specific patterns, auth rules
│   └── frontend/
│       └── AGENTS.md        ← ~2 KiB: component conventions, Vitest setup
└── infra/
    └── AGENTS.md            ← ~2 KiB: Terraform patterns, never-touch rules
```

When Codex works in `src/api/`, it loads the repo root + `src/api/AGENTS.md` — a focused 6 KiB chain, not the entire 10 KiB tree.

---

## AGENTS.md as a Universal Standard

`AGENTS.md` is not Codex-only. It is now an **open standard** stewarded by the Agentic AI Foundation under the Linux Foundation,[^5] with support across:

| Tool | AGENTS.md support |
|------|------------------|
| OpenAI Codex | Native |
| Amp | Native |
| Jules (Google) | Native |
| Claude Code | Reads AGENTS.md as fallback if no CLAUDE.md present[^6] ⚠️ [unverified] |
| Cursor | Via AGENTS.md plugin |
| Factory | Native |

**Practical benefit:** Write one `AGENTS.md` per repo. It guides all AI tools your team uses, without maintaining parallel `CLAUDE.md` + `AGENTS.md` + `cursor-rules` files.

---

## Debugging the Active Instruction Chain

### Check which files are loaded

```bash
codex --cd services/payments --ask-for-approval never \
  "List the AGENTS.md files you loaded, in order, with their paths."
```

### View the session log

```bash
cat ~/.codex/log/codex-tui.log | grep -i agents
```

The log shows exactly which files Codex found and concatenated. Useful when an override isn't taking effect.[^1]

### Common gotchas

| Symptom | Likely cause |
|---------|-------------|
| Override not applying | Wrong filename: use `AGENTS.override.md`, not `AGENTS-override.md`[^1] |
| Old rules persisting | You edited a parent `AGENTS.md` but a closer `AGENTS.override.md` is still winning |
| File silently ignored | File is empty, or combined chain already hit `project_doc_max_bytes`[^1] |
| Rules applying in wrong directory | Codex resolves from the Git root, not from where you `cd` — run from the target dir[^1] |

---

## Proposed: Named Agent Variants (`--agents` flag)

A [GitHub issue](https://github.com/openai/codex/issues/10067) proposes a `--agents` flag for switching between named agent configurations:[^7]

```bash
codex --agents review "Review this PR for security issues"
# Loads: AGENTS.review.md (review-focused instructions)

codex --agents refactor "Clean up the checkout module"
# Loads: AGENTS.refactor.md (refactor-focused instructions)
```

**Not yet shipped**[^7] — track the issue if you want role-specific agent configurations without manually swapping override files.

---

## Quick Reference

```toml
# ~/.codex/config.toml
[agent]
project_doc_max_bytes = 65536                              # raise size limit (bytes)
project_doc_fallback_filenames = ["CONTRIBUTING.md"]       # additional filenames to check
```

```
Priority order at each directory level:
  AGENTS.override.md  >  AGENTS.md  >  fallback_filenames
```

```
Files concatenate root → cwd; cwd content wins conflicts.
~/.codex/AGENTS.override.md wins everything globally.
```

---

## Citations

[^1]: OpenAI. "Custom instructions with AGENTS.md." OpenAI Developers documentation. <https://developers.openai.com/codex/guides/agents-md>

[^2]: OpenAI. "Advanced Configuration." OpenAI Developers documentation — describes `project_doc_max_bytes` (32 KiB default) and `project_doc_fallback_filenames`. <https://developers.openai.com/codex/config-advanced>

[^3]: openai/codex GitHub issue #4376. "Add an option to fall back to an alternative filename for the project-level instruction file (AGENTS.md) if it's not present." Closed 2025-10-01 via PR #4544. <https://github.com/openai/codex/issues/4376>

[^4]: openai/codex GitHub issue #13386. "AGENTS.md is silently truncated and instructions near the end ignored." Reports silent truncation at 32 KB with no warning in the TUI, observed on version 0.108.0-alpha.3. <https://github.com/openai/codex/issues/13386>

[^5]: Linux Foundation. "Linux Foundation Announces the Formation of the Agentic AI Foundation (AAIF), Anchored by New Project Contributions Including Model Context Protocol (MCP), goose and AGENTS.md." Press release, December 9, 2025. <https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation>

[^6]: Third-party source (deployhq.com) states Claude Code reads AGENTS.md as a fallback when no CLAUDE.md is present. Anthropic's official Claude Code documentation does not explicitly confirm this behaviour — treat with caution. <https://www.deployhq.com/blog/ai-coding-config-files-guide>

[^7]: openai/codex GitHub issue #10067. "--agents flag for switching between named agent configurations." Status: open as of 2026-03-26. <https://github.com/openai/codex/issues/10067>
