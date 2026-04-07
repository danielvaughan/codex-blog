# codex-blog

Public Jekyll blog at <https://codex.danielvaughan.com>. Articles and
sketchnotes are mirrored automatically from the private `codex-resources`
repository.

## How content is published

1. You write or edit an article in `codex-resources/articles/` and push to main.
2. A GitHub Action in `codex-resources` runs `scripts/sync-to-codex-blog.sh`,
   which `rsync`s the articles into `_posts/` and the sketchnote PNGs into
   `sketchnotes/articles/`, rewrites image paths from
   `/codex-resources/sketchnotes/` → `/sketchnotes/`, and pushes the result to
   this repo.
3. GitHub Pages rebuilds and publishes the site.

You can also run the sync manually from the **Actions** tab in
`codex-resources` → **sync-to-codex-blog** → **Run workflow**.

## Do not edit synced folders

These folders are overwritten on every sync run:

- `_posts/` — articles
- `sketchnotes/articles/` — sketchnote PNGs

A `SYNCED-DO-NOT-EDIT.md` marker file in each folder is preserved by the sync
script. Anything else you add or edit there will be wiped.

## Hand-authored files

These you can edit normally:

- `_config.yml` — site config
- `_data/navigation.yml` — top navigation
- `index.html` — home page (must be at root for pagination)
- `_pages/tags.md` — tag-index page
- `assets/css/main.scss` — theme overrides
- `CNAME` — custom domain
- `Gemfile` — Ruby gems (run `bundle install` after editing)

## Local development

```bash
bundle install
bundle exec jekyll serve
```

Then open <http://localhost:4000>.

The first build will be slow because it pulls the remote `minimal-mistakes`
theme. Subsequent builds use the cache.

## One-time setup checklist

If you are setting this repo up from scratch, work through this in order:

- [ ] Create the public GitHub repo `danielvaughan/codex-blog`.
- [ ] Push this local repo to it: `git remote add origin
      git@github.com:danielvaughan/codex-blog.git && git push -u origin main`.
- [ ] In **Settings → Pages**, set source to **Deploy from a branch**, branch
      `main`, folder `/`. Set custom domain to `codex.danielvaughan.com`.
- [ ] Add a DNS `CNAME` record `codex → danielvaughan.github.io.` (note the
      trailing dot) at your DNS provider.
- [ ] Wait for DNS propagation, then enable **Enforce HTTPS** in
      Settings → Pages.
- [ ] Create a fine-grained Personal Access Token scoped to **only** the
      `danielvaughan/codex-blog` repo with **Contents: Read and write**
      permission. Save it as the `CODEX_BLOG_DEPLOY_TOKEN` secret in the
      `codex-resources` repo (Settings → Secrets and variables → Actions).
- [ ] In `codex-resources`, run the `sync-to-codex-blog` workflow manually
      from the **Actions** tab to perform the first sync.

## Architecture

See [`docs/superpowers/specs/2026-04-07-codex-blog-design.md`](docs/superpowers/specs/2026-04-07-codex-blog-design.md).
