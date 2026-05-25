# AGENTS.md — codex-blog

## How this repo is updated

This repo is almost exclusively updated by the sync workflow in `codex-resources`.
Direct edits to `_posts/` or `sketchnotes/articles/` will be overwritten on the
next sync. If you need to fix article content, fix it upstream in
`codex-resources/articles/` instead.

The sync script (`codex-resources/scripts/sync-to-codex-blog.sh`) handles:
- Copying articles and sketchnotes
- Rewriting image paths and cross-reference links
- Escaping Liquid syntax in code fences (via `escape-liquid-in-fences.py`)
- Injecting `last_modified_at` from git history
- Stripping/normalising front matter

## What you can edit here

Only template-level and infrastructure files:
- `_config.yml`, `_data/`, `_pages/`, `_includes/`, `_layouts/`
- `assets/css/main.scss`
- `index.html`
- `.github/workflows/deploy.yml`
- `Gemfile` / `Gemfile.lock`
- `CNAME`, `README.md`, this file

## Naming

- Use **"Codex"** rather than "Codex CLI" in all user-facing copy (site title, descriptions, meta tags, topic hub text, navigation).
- "Codex CLI" is acceptable only when disambiguating from OpenAI's deprecated Codex model or when referring specifically to the CLI binary itself.

## Site framing

- This site is a **knowledge base**, not a blog. Use "knowledge base", "reference", or "articles" — never "blog" or "blog post" in user-facing text.
- The site title is **"Codex Knowledge Base"**.

## Fixing build failures

If the Jekyll deploy fails due to Liquid syntax errors, the cause is almost
always `{{` or `{%` inside code fences in an article. The fix belongs upstream:
either edit the article in `codex-resources`, or improve
`codex-resources/scripts/escape-liquid-in-fences.py` to handle the new pattern.
Do not patch `_posts/` directly — it will be overwritten.
