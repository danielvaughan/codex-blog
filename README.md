# Codex Knowledge Base

[![pages-build-deployment](https://github.com/danielvaughan/codex-blog/actions/workflows/pages/pages-build-deployment/badge.svg)](https://github.com/danielvaughan/codex-blog/actions/workflows/pages/pages-build-deployment)
[![Deploy Jekyll to GitHub Pages](https://github.com/danielvaughan/codex-blog/actions/workflows/deploy.yml/badge.svg)](https://github.com/danielvaughan/codex-blog/actions/workflows/deploy.yml)

Public Jekyll site at <https://codex.danielvaughan.com>. This repo should
not be edited manually — content is published automatically from the
private `codex-resources` repository.

## How content arrives

1. Articles are written in `codex-resources/articles/` and pushed to main.
2. A GitHub Action in `codex-resources` runs `scripts/sync-to-codex-blog.sh`,
   which syncs articles into `_posts/`, sketchnotes into
   `sketchnotes/articles/`, rewrites image paths, escapes Liquid syntax in
   code fences, and pushes the result here.
3. The `deploy.yml` workflow builds Jekyll (with full plugin support) and
   deploys to GitHub Pages.

## Do not edit

These folders are overwritten on every sync:

- `_posts/` — articles
- `sketchnotes/articles/` — sketchnote PNGs

A `SYNCED-DO-NOT-EDIT.md` marker in each folder is preserved by the sync
script. Everything else will be wiped.

## Exceptions (hand-authored)

These files live in this repo and can be edited directly:

- `_config.yml` — site config
- `_data/navigation.yml` — top navigation
- `index.html` — home page (must be at root for pagination)
- `_pages/tags.md` — tag-index page
- `assets/css/main.scss` — theme overrides
- `CNAME` — custom domain
- `Gemfile` — Ruby gems (run `bundle install` after editing)
- `.github/workflows/deploy.yml` — Jekyll build and deploy

## Local development

```bash
bundle install
bundle exec jekyll serve
```

Then open <http://localhost:4000>.

## Architecture

See [`docs/superpowers/specs/2026-04-07-codex-blog-design.md`](docs/superpowers/specs/2026-04-07-codex-blog-design.md).
