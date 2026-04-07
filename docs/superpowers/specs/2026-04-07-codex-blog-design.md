---
title: codex-blog — public Jekyll blog mirrored from codex-resources
date: 2026-04-07
status: design
---

# codex-blog Design

## Motivation

`codex-resources` is a private Jekyll site holding articles, sketchnotes, book
material, marketing assets, presentations, and internal notes. It cannot be made
public because of the surrounding non-article content.

`codex-blog` is a new, **public** GitHub Pages site that exposes only the
articles and their sketchnotes from `codex-resources`. It is the public face of
that body of work; `codex-resources` remains the private source of truth where
articles are written and edited.

## Goals

- A public Jekyll blog at `https://codex.danielvaughan.com` containing all 164
  articles and ~30+ sketchnotes from `codex-resources`.
- Zero manual copy-paste. Articles flow from `codex-resources` → `codex-blog`
  via a GitHub Action that runs on commit and on demand.
- Familiar look: same `minimal-mistakes` theme as `codex-resources`, trimmed
  down to a blog-only layout (no doc sidebar).
- Reverse-chronological home page, tag index page, RSS feed, search.

## Non-goals

- Two-way sync. `codex-resources` is the source of truth; nothing flows back.
- Selective publishing (drafts, scheduled posts, per-tag filtering). Everything
  in `codex-resources/articles/*.md` is published.
- Conflict resolution if a human edits an auto-synced file. The sync uses
  `rsync --delete`; manual edits will be wiped on the next run. Synced folders
  carry a `SYNCED-DO-NOT-EDIT.md` marker.
- Automated DNS or GitHub repo settings. DNS records and the Pages "custom
  domain" toggle are manual one-time steps documented in the README.

## Architecture

```
┌────────────────────────┐    push to main /     ┌──────────────────────┐
│  codex-resources       │    workflow_dispatch  │  GitHub Action       │
│  (private)             │ ────────────────────▶ │  sync-to-codex-blog  │
│                        │                       │  (in codex-resources)│
│  articles/*.md         │                       └──────────┬───────────┘
│  sketchnotes/articles/ │                                  │
└────────────────────────┘                                  │ rsync + path
                                                            │ rewrite +
                                                            │ git push
                                                            ▼
┌────────────────────────┐    GitHub Pages       ┌──────────────────────┐
│  codex-blog            │    rebuild            │  codex.danielvaughan │
│  (public)              │ ────────────────────▶ │  .com                │
│                        │                       │                      │
│  _posts/*.md (synced)  │                       │  Public blog         │
│  sketchnotes/articles/ │                       │                      │
│  (synced)              │                       │                      │
│  + theme, config, nav  │                       │                      │
└────────────────────────┘                       └──────────────────────┘
```

### Source-of-truth split

- **Hand-authored in codex-blog** (rare changes): `_config.yml`, `Gemfile`,
  `CNAME`, `_data/navigation.yml`, `index.html` (the paginated home page —
  must live at root for jekyll-paginate v1), `_pages/tags.md`,
  `assets/css/main.scss`, `README.md`, `.gitignore`, the sync workflow
  documentation.
- **Auto-synced into codex-blog** (overwritten on every run, never edited by
  hand): `_posts/*.md`, `sketchnotes/articles/*.png`. A `SYNCED-DO-NOT-EDIT.md`
  marker file in each folder makes that explicit.

## codex-blog repo layout

```
codex-blog/
├── _config.yml                  # Jekyll config (remote theme, blog skin)
├── _data/
│   └── navigation.yml           # Top nav: Home, Tags
├── index.html                   # layout: home, paginated post list
├── _pages/
│   └── tags.md                  # tag index, permalink: /tags/
├── _posts/                      # AUTO-SYNCED — never edit by hand
│   ├── SYNCED-DO-NOT-EDIT.md
│   └── 2026-03-26-*.md          # 164 files synced from codex-resources
├── sketchnotes/
│   └── articles/                # AUTO-SYNCED — never edit by hand
│       ├── SYNCED-DO-NOT-EDIT.md
│       └── 2026-03-26-*.png
├── assets/
│   └── css/main.scss            # Theme overrides (one file)
├── CNAME                        # codex.danielvaughan.com
├── Gemfile                      # github-pages gem
├── README.md                    # Sync flow + DNS setup notes
├── .gitignore                   # _site/, .jekyll-cache/, vendor/, .bundle/
└── docs/
    └── superpowers/specs/
        └── 2026-04-07-codex-blog-design.md   # this file
```

### Why `_posts/` instead of `articles/`

`codex-resources` keeps articles in a regular `articles/` folder because the
site is broader than a blog. For a blog-only site, putting them in Jekyll's
`_posts/` collection unlocks chronological iteration via `site.posts`,
automatic tag archives, RSS feed via `jekyll-feed`, dated permalinks, and
pagination — all for free. The filenames already match the `YYYY-MM-DD-slug.md`
convention, so this is purely a destination-folder choice in the sync script.

## _config.yml (key settings)

```yaml
title: "Codex Blog"
description: "Articles on agentic software engineering with Codex CLI"
url: "https://codex.danielvaughan.com"
baseurl: ""
repository: "danielvaughan/codex-blog"

remote_theme: mmistakes/minimal-mistakes@4.26.2
minimal_mistakes_skin: "air"

permalink: /:year/:month/:day/:title/
timezone: Europe/London
markdown: kramdown

paginate: 10
paginate_path: "/page:num/"

search: true
search_full_content: true

plugins:
  - jekyll-remote-theme
  - jekyll-seo-tag
  - jekyll-sitemap
  - jekyll-feed
  - jekyll-paginate
  - jekyll-include-cache

author:
  name: "Daniel Vaughan"
  bio: "Building expertise in agentic software engineering with Codex CLI"
  links:
    - label: "GitHub"
      icon: "fab fa-fw fa-github"
      url: "https://github.com/danielvaughan"

defaults:
  - scope:
      path: ""
      type: posts
    values:
      layout: single
      author_profile: true
      read_time: true
      share: true
      related: true
      comments: false
      toc: true
      toc_sticky: true
  - scope:
      path: "_pages"
      type: pages
    values:
      layout: single
      author_profile: true

exclude:
  - Gemfile
  - Gemfile.lock
  - vendor/
  - .bundle/
  - docs/
```

## Home page

`index.html` at the repo root with `layout: home`. Renders `paginator.posts`
showing title, date, tags, excerpt, read-time. 10 posts per page. Pagination
via `jekyll-paginate` v1, which only paginates files literally named
`index.html` at the matching path — that's why the home page lives at the
root, not in `_pages/`.

## Tags page

`_pages/tags.md` with `permalink: /tags/`. Iterates `site.tags` to render an
alphabetical list of tags, each with a count and a list of posts under it. No
per-tag URLs (no `/tags/codex-cli/` archive). One file, one page. If per-tag
archives are wanted later, minimal-mistakes supports
`tag_archive: type: liquid`.

## Navigation

`_data/navigation.yml`:

```yaml
main:
  - title: "Home"
    url: /
  - title: "Tags"
    url: /tags/
```

No About page (decided in brainstorming).

## Theme overrides

Single file: `assets/css/main.scss`. Imports the minimal-mistakes theme then
sets a small set of variables (max content width, line-height, link colour).
No custom layouts unless something visibly breaks. The skin is `air`; alternate
skins can be tried by changing the one config value.

## Custom domain setup

**In repo:**
- `CNAME` file at root containing exactly `codex.danielvaughan.com` (no
  protocol, no trailing slash).
- `_config.yml` has `url: "https://codex.danielvaughan.com"` and an empty
  `baseurl`.

**Manual one-time steps (out of repo):**
- DNS: add a `CNAME` record for `codex` pointing to `danielvaughan.github.io.`
  (trailing dot).
- GitHub repo settings → Pages: source = "Deploy from a branch", branch =
  `main`, folder = `/`. Set custom domain to `codex.danielvaughan.com`. After
  DNS propagates, enable "Enforce HTTPS".

These manual steps will be listed in the README and called out in the
implementation plan as user-action items.

## Sync GitHub Action

**Lives in:** `codex-resources/.github/workflows/sync-to-codex-blog.yml`
(NOT in codex-blog).

**Triggers:**
- `push` to `main` filtered on paths `articles/**` and
  `sketchnotes/articles/**`.
- `workflow_dispatch` for manual runs.

**Steps:**
1. Checkout codex-resources (default token).
2. Checkout codex-blog into a sub-path, using a fine-grained PAT stored as
   secret `CODEX_BLOG_DEPLOY_TOKEN`. The PAT is scoped to the codex-blog repo
   only with `contents: write`.
3. Run `scripts/sync-to-codex-blog.sh` (lives in codex-resources). Env:
   - `CODEX_BLOG_DIR` — path to checked-out codex-blog.
   - `SOURCE_SHA` — `${{ github.sha }}`.
4. The script handles its own commit and push.

**Why a PAT and not `GITHUB_TOKEN`:** the built-in token can't push to a
different repo. A fine-grained PAT scoped only to codex-blog with
`contents: write` is the least-privilege option.

## Sync script

**Lives at:** `codex-resources/scripts/sync-to-codex-blog.sh`.

**Inputs (env):**
- `CODEX_BLOG_DIR` — path to checked-out codex-blog working tree.
- `SOURCE_SHA` — codex-resources commit SHA, used in commit message.

**Steps:**

1. **Validate inputs.** Fail loudly if `CODEX_BLOG_DIR` is unset or doesn't
   exist, or if `articles/` or `sketchnotes/articles/` are missing in
   codex-resources.

2. **Sync articles** to `_posts/`:
   ```
   rsync -av --delete \
     --include='*.md' \
     --exclude='SYNCED-DO-NOT-EDIT.md' \
     --exclude='*' \
     articles/ "$CODEX_BLOG_DIR/_posts/"
   ```
   The `--exclude='SYNCED-DO-NOT-EDIT.md'` prevents the marker file from being
   deleted.

3. **Sync sketchnotes** to `sketchnotes/articles/`:
   ```
   rsync -av --delete \
     --include='*.png' \
     --exclude='SYNCED-DO-NOT-EDIT.md' \
     --exclude='*' \
     sketchnotes/articles/ "$CODEX_BLOG_DIR/sketchnotes/articles/"
   ```

4. **Rewrite sketchnote paths** in synced posts:
   ```
   find "$CODEX_BLOG_DIR/_posts" -name '*.md' -exec \
     sed -i.bak 's|/codex-resources/sketchnotes/|/sketchnotes/|g' {} \;
   find "$CODEX_BLOG_DIR/_posts" -name '*.bak' -delete
   ```
   This rewrites `![Sketchnote](/codex-resources/sketchnotes/articles/foo.png)`
   to `![Sketchnote](/sketchnotes/articles/foo.png)`.

5. **Detect changes.** `cd "$CODEX_BLOG_DIR" && git status --porcelain`. If
   empty, print "nothing to sync" and exit 0.

6. **Commit and push.** Configure bot identity (`codex-blog-sync[bot]`),
   `git add -A`, commit message:
   `chore: sync from codex-resources@<short-sha>`, then `git push`.

**Failure modes handled:**
- Missing source directories → fail loudly with clear message.
- Missing `CODEX_BLOG_DIR` → fail loudly.
- `git push` rejected (codex-blog moved underneath us) → fail loudly; the
  action surfaces it as a red CI run; re-run after manual investigation.

**Failure modes deliberately not handled (YAGNI):**
- Conflict resolution if a human edited a synced file.
- Selective sync by tag, draft state, or date.
- Front matter normalisation. Articles already use `layout: single` which
  minimal-mistakes accepts; the date is in the filename which Jekyll uses for
  posts. If a future article is missing required front matter, the script
  fails fast and the issue is fixed in codex-resources.

**Local testing:** the script can be run by hand from a codex-resources
checkout with both repos cloned side by side, before being wired into the
action.

## What gets implemented where

### In codex-blog (this repo)

- `_config.yml`, `Gemfile`, `CNAME`, `.gitignore`
- `_data/navigation.yml`
- `index.html`, `_pages/tags.md`
- `assets/css/main.scss` (theme overrides)
- `_posts/SYNCED-DO-NOT-EDIT.md` (marker)
- `sketchnotes/articles/SYNCED-DO-NOT-EDIT.md` (marker)
- `README.md` (explains sync flow, DNS setup, manual repo steps)
- This design doc at `docs/superpowers/specs/2026-04-07-codex-blog-design.md`

### In codex-resources (separate PR/commit)

- `scripts/sync-to-codex-blog.sh`
- `.github/workflows/sync-to-codex-blog.yml`
- A note in `codex-resources/AGENTS.md` (or wherever) explaining the sync flow.

### Manual one-time steps (user actions)

- Initialise codex-blog as a git repo and push to `github.com/danielvaughan/codex-blog`.
- Add DNS `CNAME` record `codex → danielvaughan.github.io.`.
- In codex-blog repo settings → Pages: set custom domain, enable HTTPS once
  DNS propagates.
- Create a fine-grained PAT scoped to codex-blog with `contents: write`, store
  it in codex-resources secrets as `CODEX_BLOG_DEPLOY_TOKEN`.
- First run: trigger the sync workflow manually via `workflow_dispatch` to
  populate codex-blog.

## Open questions

None remaining at design time. Implementation plan will sequence the work and
mark each manual step explicitly.