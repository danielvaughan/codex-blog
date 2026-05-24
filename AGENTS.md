# AGENTS.md — codex-blog

## Naming

- Use **"Codex"** rather than "Codex CLI" in all user-facing copy (site title, descriptions, meta tags, topic hub text, navigation).
- "Codex CLI" is acceptable only when disambiguating from OpenAI's deprecated Codex model or when referring specifically to the CLI binary itself.

## Site framing

- This site is a **knowledge base**, not a blog. Use "knowledge base", "reference", or "articles" — never "blog" or "blog post" in user-facing text.
- The site title is **"Codex Knowledge Base"**.

## Content

- Articles are synced from `codex-resources` via `scripts/sync-to-codex-blog.sh`. Never hand-edit files in `_posts/` — they will be overwritten on next sync.
- Template-level changes (layouts, includes, config, styles) are hand-authored in this repo.
