# AGENTS.md — codex-blog

## This repository is public

Everything committed here is published. Name the private `codex-resources` repository only as the source of the
articles; never copy its private content, notes or other private repositories' details into this repository.

## Synced posts are not hand-edited

`_posts/` and `sketchnotes/articles/` are written by `codex-resources`' sync script
(`scripts/sync-to-codex-blog.sh`), which deletes and overwrites them on every sync. The `SYNCED-DO-NOT-EDIT.md`
marker in each folder says so. Do not hand-edit anything in them; fix articles upstream.

## Keep the README current

[README.md](README.md) is the owner's guide to this repository, written with the `repo-readme` skill (zx-plugins
`repo-guide` plugin). After every change you commit, reread the README sections it touches and update them in the
same commit. Run the skill's `check_readme.py README.md` before committing, and change the "Last reviewed in full"
date only after a full review.

## Keep GitHub's main current

This machine can fail or be handed over at any time, so GitHub's `main` must hold every change (owner,
21 November 2026). Work that exists only on this machine can be lost.

- Commit each finished change as soon as its checks pass, then push it to `origin main` in the same session:
  `git pull --rebase origin main && git push origin main`. The sync bot pushes here several times a day, so always
  rebase first.
- Work on `main`. If you must use another branch, merge it into `main` and push as soon as the work is checked,
  and delete the branch once merged.
- Before you finish a session, run `git status -sb`: it must show nothing ahead of `origin/main`. Push anything
  that is.
- If a push is rejected because `origin/main` moved, rebase onto it and push again. Never force-push `main`, and
  never rewrite history that is already pushed.
- Never push secrets, credentials or local runtime data: the commit hooks and `.gitignore` decide what belongs in
  Git.

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
