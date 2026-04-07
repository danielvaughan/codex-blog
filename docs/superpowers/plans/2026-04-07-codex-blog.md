# codex-blog Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up `codex-blog`, a public Jekyll blog at `codex.danielvaughan.com` whose articles and sketchnotes are mirrored from the private `codex-resources` repo by a GitHub Action.

**Architecture:** Jekyll site using the `minimal-mistakes` remote theme. Articles live in `_posts/` and sketchnote PNGs in `sketchnotes/articles/` — both folders are auto-populated by `scripts/sync-to-codex-blog.sh` (lives in codex-resources) which is triggered by a GitHub Action on push to main and on `workflow_dispatch`. The sync uses `rsync --delete`, rewrites image paths (`/codex-resources/sketchnotes/` → `/sketchnotes/`), and pushes to codex-blog with a fine-grained PAT.

**Tech Stack:** Jekyll 3.x via the `github-pages` gem, `minimal-mistakes` remote theme, GitHub Pages with custom domain, GitHub Actions, rsync, sed, bash.

**Spec:** [`docs/superpowers/specs/2026-04-07-codex-blog-design.md`](../specs/2026-04-07-codex-blog-design.md)

---

## File Structure

### Files to create in `codex-blog` (this repo)

| Path | Purpose |
|---|---|
| `.gitignore` | Ignore Jekyll build output, bundler caches, OS junk |
| `Gemfile` | Pin `github-pages` gem so local builds match Pages |
| `_config.yml` | Jekyll site config (theme, plugins, author, defaults) |
| `_data/navigation.yml` | Top nav items (Home, Tags) |
| `index.html` | Home page — lists `site.posts` reverse-chrono with pagination |
| `_pages/tags.md` | Tag index page — lists tags + posts under each |
| `assets/css/main.scss` | SCSS overrides for theme |
| `_posts/SYNCED-DO-NOT-EDIT.md` | Marker file in synced folder (preserved by sync) |
| `sketchnotes/articles/SYNCED-DO-NOT-EDIT.md` | Marker file in synced folder (preserved by sync) |
| `CNAME` | Custom-domain pointer for GitHub Pages |
| `README.md` | Documents sync flow + manual DNS/Pages/PAT steps |

The design spec already lives at `docs/superpowers/specs/2026-04-07-codex-blog-design.md` and stays put.

### Files to create in `codex-resources` (separate repo, separate commit)

| Path | Purpose |
|---|---|
| `scripts/sync-to-codex-blog.sh` | The sync script (rsync + sed + git commit/push) |
| `scripts/test-sync-to-codex-blog.sh` | Test harness using fixtures and asserts |
| `.github/workflows/sync-to-codex-blog.yml` | GitHub Action that invokes the sync script |

### Manual one-time user actions (not implemented in code)

- Initialise codex-blog as a git repo, push to `github.com/danielvaughan/codex-blog`.
- Create the public GitHub repo `danielvaughan/codex-blog`.
- Add DNS `CNAME` record `codex → danielvaughan.github.io.`.
- In codex-blog repo settings → Pages: set custom domain, enable HTTPS once DNS propagates.
- Create a fine-grained PAT scoped to codex-blog with `contents: write`, store as secret `CODEX_BLOG_DEPLOY_TOKEN` in codex-resources.
- First run: trigger the sync workflow manually via `workflow_dispatch`.

These are listed in `README.md` as a checklist, and called out at the end of this plan.

---

## Phase 1 — codex-blog skeleton

### Task 1: Initialise the git repo and add .gitignore

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/.gitignore`

- [ ] **Step 1: Init git repo**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git init -b main
```

Expected: `Initialized empty Git repository in .../codex-blog/.git/`

- [ ] **Step 2: Create .gitignore**

```
_site/
.jekyll-cache/
.jekyll-metadata
vendor/
.bundle/
*.gem
.DS_Store
Gemfile.lock
```

- [ ] **Step 3: Verify file**

```bash
cat /Users/danielvaughan/Development/git/codex-blog/.gitignore
```

Expected: prints the contents above.

- [ ] **Step 4: Commit**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git add .gitignore docs/
git commit -m "chore: initial commit with .gitignore and design spec"
```

Expected: one commit created with `.gitignore` + the design spec already present at `docs/superpowers/specs/2026-04-07-codex-blog-design.md`.

---

### Task 2: Add the Gemfile

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/Gemfile`

- [ ] **Step 1: Write Gemfile**

```ruby
source "https://rubygems.org"

# GitHub Pages — pinned gem set used by github.io builds
gem "github-pages", group: :jekyll_plugins

# Required for `bundle exec jekyll serve` on Ruby 3+
gem "webrick"

# Plugins explicitly used in _config.yml
group :jekyll_plugins do
  gem "jekyll-remote-theme"
  gem "jekyll-seo-tag"
  gem "jekyll-sitemap"
  gem "jekyll-feed"
  gem "jekyll-paginate"
  gem "jekyll-include-cache"
end
```

- [ ] **Step 2: Install gems**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
bundle install
```

Expected: `Bundle complete!`. Creates `Gemfile.lock` (which is gitignored).

If `bundle` is not installed: `gem install bundler` first. If Ruby itself is missing or too old, install via Homebrew: `brew install ruby` and add it to PATH.

- [ ] **Step 3: Commit**

```bash
git add Gemfile
git commit -m "chore: add Gemfile with github-pages and required plugins"
```

---

### Task 3: Write _config.yml

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/_config.yml`

- [ ] **Step 1: Write _config.yml**

```yaml
# Site
title: "Codex Blog"
description: "Articles on agentic software engineering with Codex CLI"
url: "https://codex.danielvaughan.com"
baseurl: ""
repository: "danielvaughan/codex-blog"

# Theme
remote_theme: mmistakes/minimal-mistakes@4.26.2
minimal_mistakes_skin: "air"

# Build
permalink: /:year/:month/:day/:title/
timezone: Europe/London
markdown: kramdown

# Pagination
paginate: 10
paginate_path: "/page:num/"

# Search
search: true
search_full_content: true

# Plugins (also listed in Gemfile)
plugins:
  - jekyll-remote-theme
  - jekyll-seo-tag
  - jekyll-sitemap
  - jekyll-feed
  - jekyll-paginate
  - jekyll-include-cache

# Author
author:
  name: "Daniel Vaughan"
  bio: "Building expertise in agentic software engineering with Codex CLI"
  links:
    - label: "GitHub"
      icon: "fab fa-fw fa-github"
      url: "https://github.com/danielvaughan"

# Defaults
defaults:
  # All posts
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
  # All hand-authored pages
  - scope:
      path: "_pages"
      type: pages
    values:
      layout: single
      author_profile: true

# Include _pages so Jekyll picks them up
include:
  - _pages

# Exclude
exclude:
  - Gemfile
  - Gemfile.lock
  - vendor/
  - .bundle/
  - docs/
  - README.md
  - "*.gem"
  - "*.gemspec"
```

- [ ] **Step 2: Verify Jekyll can build with no posts yet**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
bundle exec jekyll build
```

Expected: build succeeds with warnings about empty `_posts/` (acceptable). Output goes to `_site/`. If it fails because `_pages` doesn't exist yet, that's fine — Tasks 4 and 5 create those files.

- [ ] **Step 3: Commit**

```bash
git add _config.yml
git commit -m "chore: add _config.yml with minimal-mistakes remote theme"
```

---

### Task 4: Add navigation data

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/_data/navigation.yml`

- [ ] **Step 1: Write navigation.yml**

```yaml
main:
  - title: "Home"
    url: /
  - title: "Tags"
    url: /tags/
```

- [ ] **Step 2: Commit**

```bash
git add _data/navigation.yml
git commit -m "chore: add top navigation (Home, Tags)"
```

---

### Task 5: Add home and tags pages

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/index.html`
- Create: `/Users/danielvaughan/Development/git/codex-blog/_pages/tags.md`

The home page MUST be at the repo root and named `index.html` so that jekyll-paginate v1 (the version shipped with the github-pages gem) recognises and paginates it. A `_pages/home.md` with a permalink will not be paginated by v1.

- [ ] **Step 1: Write index.html**

```html
---
title: "Codex Blog"
layout: home
author_profile: true
---
```

The `layout: home` is provided by minimal-mistakes; it renders `paginator.posts` automatically when `paginate` is set in `_config.yml`. The body is empty — the layout supplies the post list.

- [ ] **Step 2: Write tags.md**

```markdown
---
title: "Tags"
permalink: /tags/
layout: single
author_profile: true
toc: true
toc_label: "Tags"
toc_sticky: true
---

{% assign tags_sorted = site.tags | sort %}

{% for tag in tags_sorted %}
  {% assign tag_name = tag | first %}
  {% assign tag_posts = tag | last %}

## {{ tag_name }}

  <ul>
  {% for post in tag_posts %}
    <li>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      — <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%Y-%m-%d" }}</time>
    </li>
  {% endfor %}
  </ul>
{% endfor %}
```

- [ ] **Step 3: Verify build**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
bundle exec jekyll build
```

Expected: build succeeds. `_site/index.html` and `_site/tags/index.html` exist. They will be empty of posts (since `_posts/` is still empty) but the pages render.

- [ ] **Step 4: Verify pages rendered**

```bash
ls /Users/danielvaughan/Development/git/codex-blog/_site/
ls /Users/danielvaughan/Development/git/codex-blog/_site/tags/
```

Expected: both directories exist; `_site/index.html` and `_site/tags/index.html` are present.

- [ ] **Step 5: Commit**

```bash
git add index.html _pages/tags.md
git commit -m "feat: add home (index.html) and tags pages"
```

---

### Task 6: Add SCSS overrides

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/assets/css/main.scss`

- [ ] **Step 1: Write main.scss**

```scss
---
# Front matter required so Jekyll processes this file as SCSS
---

@charset "utf-8";

// Skin selection — picks up _config.yml's minimal_mistakes_skin value
@import "minimal-mistakes/skins/{{ site.minimal_mistakes_skin | default: 'default' }}";

// Core theme
@import "minimal-mistakes";

// ----- Local overrides -----

// Slightly more comfortable line-height for long-form reading
.page__content p,
.page__content li,
.page__content dl {
  line-height: 1.7;
}

// Wider max content width on large screens
@include breakpoint($large) {
  .page {
    padding-right: 0;
  }
}
```

- [ ] **Step 2: Verify build still succeeds**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
bundle exec jekyll build
```

Expected: build succeeds. `_site/assets/css/main.css` is generated.

- [ ] **Step 3: Commit**

```bash
git add assets/css/main.scss
git commit -m "feat: add SCSS overrides for line-height and content width"
```

---

### Task 7: Add CNAME for custom domain

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/CNAME`

- [ ] **Step 1: Write CNAME**

```
codex.danielvaughan.com
```

No protocol, no trailing slash, no newline tricks. One line.

- [ ] **Step 2: Verify**

```bash
cat /Users/danielvaughan/Development/git/codex-blog/CNAME
```

Expected: prints `codex.danielvaughan.com`.

- [ ] **Step 3: Commit**

```bash
git add CNAME
git commit -m "chore: add CNAME for codex.danielvaughan.com"
```

---

### Task 8: Add SYNCED-DO-NOT-EDIT marker files

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/_posts/SYNCED-DO-NOT-EDIT.md`
- Create: `/Users/danielvaughan/Development/git/codex-blog/sketchnotes/articles/SYNCED-DO-NOT-EDIT.md`

- [ ] **Step 1: Write _posts marker**

```markdown
---
sitemap: false
published: false
---

# Do not edit files in this directory by hand

This folder is auto-synced from the private `codex-resources` repository by the
`sync-to-codex-blog` GitHub Action. Any manual changes here will be overwritten
on the next sync run.

To change a post, edit it in `codex-resources/articles/` and push to main.
```

The `published: false` front matter prevents Jekyll from rendering this marker as a public post.

- [ ] **Step 2: Write sketchnotes marker**

```markdown
# Do not edit files in this directory by hand

This folder is auto-synced from the private `codex-resources` repository by the
`sync-to-codex-blog` GitHub Action. Any manual PNGs added here will be deleted
on the next sync run.

To add or change a sketchnote, put it in
`codex-resources/sketchnotes/articles/` and push to main.
```

This one isn't a post so it doesn't need front matter.

- [ ] **Step 3: Verify build still succeeds**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
bundle exec jekyll build
```

Expected: build succeeds. The marker files do not appear in `_site/`.

- [ ] **Step 4: Commit**

```bash
git add _posts/SYNCED-DO-NOT-EDIT.md sketchnotes/articles/SYNCED-DO-NOT-EDIT.md
git commit -m "chore: add SYNCED-DO-NOT-EDIT markers in auto-synced folders"
```

---

### Task 9: Write README.md

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/README.md`

- [ ] **Step 1: Write README.md**

````markdown
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
````

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add README explaining sync flow and one-time setup"
```

---

### Task 10: Verify the codex-blog skeleton builds end-to-end

- [ ] **Step 1: Clean build**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
rm -rf _site .jekyll-cache
bundle exec jekyll build
```

Expected: build succeeds with no errors. Warnings about empty `_posts/` are fine.

- [ ] **Step 2: Verify expected output files**

```bash
test -f _site/index.html && echo "index.html OK"
test -f _site/tags/index.html && echo "tags/index.html OK"
test -f _site/CNAME && echo "CNAME OK"
test -f _site/assets/css/main.css && echo "main.css OK"
test -f _site/feed.xml && echo "feed.xml OK"
```

Expected: all five lines print "OK". If `feed.xml` is missing, check that `jekyll-feed` is in `_config.yml` plugins and the Gemfile.

- [ ] **Step 3: Quick local serve sanity check**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
bundle exec jekyll serve --no-watch &
SERVE_PID=$!
sleep 5
curl -sf http://localhost:4000/ > /dev/null && echo "home OK"
curl -sf http://localhost:4000/tags/ > /dev/null && echo "tags OK"
kill $SERVE_PID
```

Expected: prints "home OK" and "tags OK", then exits.

- [ ] **Step 4: No commit needed** — this is verification only.

---

## Phase 2 — Sync script in codex-resources

These tasks operate in `/Users/danielvaughan/Development/git/codex-resources`. They are TDD: write the test fixture first, then implement the script to satisfy it.

### Task 11: Write the sync script test harness

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-resources/scripts/test-sync-to-codex-blog.sh`

- [ ] **Step 1: Write the test harness**

```bash
#!/usr/bin/env bash
# Test harness for scripts/sync-to-codex-blog.sh
#
# Creates a temp source directory mimicking codex-resources/{articles,sketchnotes/articles},
# a temp target directory mimicking codex-blog/{_posts,sketchnotes/articles},
# runs the sync script, and asserts the expected outputs.
#
# Usage: bash scripts/test-sync-to-codex-blog.sh

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
REPO_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"
SYNC_SCRIPT="$SCRIPT_DIR/sync-to-codex-blog.sh"

if [ ! -x "$SYNC_SCRIPT" ]; then
  echo "FAIL: sync script not found or not executable at $SYNC_SCRIPT" >&2
  exit 1
fi

# Set up temp dirs
WORK="$(mktemp -d -t codex-blog-sync-test.XXXXXX)"
trap 'rm -rf "$WORK"' EXIT

SRC="$WORK/src"           # plays the role of codex-resources
DST="$WORK/dst"           # plays the role of codex-blog
mkdir -p "$SRC/articles" "$SRC/sketchnotes/articles"
mkdir -p "$DST/_posts" "$DST/sketchnotes/articles"

# --- Fixtures ---

# Article that references a sketchnote with the OLD path
cat > "$SRC/articles/2026-04-07-test-post.md" <<'MD'
---
title: "Test Post"
date: 2026-04-07
tags: test
---

![Sketchnote](/codex-resources/sketchnotes/articles/2026-04-07-test-post.png)

This is a test post.
MD

# Sketchnote PNG (just a 1-byte placeholder, the bytes don't matter for the test)
printf 'x' > "$SRC/sketchnotes/articles/2026-04-07-test-post.png"

# Marker files in DST that must be preserved by --delete
cat > "$DST/_posts/SYNCED-DO-NOT-EDIT.md" <<'MD'
# marker
MD
cat > "$DST/sketchnotes/articles/SYNCED-DO-NOT-EDIT.md" <<'MD'
# marker
MD

# DST must be a git repo so the script can commit
( cd "$DST" && git init -q -b main && git add -A && \
  git -c user.email=t@t -c user.name=test commit -q -m init )

# --- Run the sync script ---
# Override the source dir via the SOURCE_DIR env var the script reads.
# SKIP_PUSH=1 because the test repo has no `origin` remote.
SKIP_PUSH=1 \
CODEX_BLOG_DIR="$DST" \
SOURCE_SHA="abc1234" \
SOURCE_DIR="$SRC" \
GIT_AUTHOR_NAME="codex-blog-sync[bot]" \
GIT_AUTHOR_EMAIL="codex-blog-sync@users.noreply.github.com" \
GIT_COMMITTER_NAME="codex-blog-sync[bot]" \
GIT_COMMITTER_EMAIL="codex-blog-sync@users.noreply.github.com" \
bash "$SYNC_SCRIPT"

# --- Assertions ---

assert_file() {
  if [ ! -f "$1" ]; then
    echo "FAIL: expected file missing: $1" >&2
    exit 1
  fi
}

assert_grep() {
  if ! grep -q "$1" "$2"; then
    echo "FAIL: pattern not found in $2: $1" >&2
    exit 1
  fi
}

assert_no_grep() {
  if grep -q "$1" "$2"; then
    echo "FAIL: forbidden pattern found in $2: $1" >&2
    exit 1
  fi
}

# Article was synced
assert_file "$DST/_posts/2026-04-07-test-post.md"

# Path was rewritten
assert_no_grep "/codex-resources/sketchnotes/" "$DST/_posts/2026-04-07-test-post.md"
assert_grep    "/sketchnotes/articles/2026-04-07-test-post.png" "$DST/_posts/2026-04-07-test-post.md"

# PNG was synced
assert_file "$DST/sketchnotes/articles/2026-04-07-test-post.png"

# Markers were preserved
assert_file "$DST/_posts/SYNCED-DO-NOT-EDIT.md"
assert_file "$DST/sketchnotes/articles/SYNCED-DO-NOT-EDIT.md"

# Commit was created
LAST_MSG="$(cd "$DST" && git log -1 --pretty=%s)"
case "$LAST_MSG" in
  "chore: sync from codex-resources@abc1234") ;;
  *) echo "FAIL: unexpected commit message: $LAST_MSG" >&2; exit 1 ;;
esac

echo "PASS: all assertions met"
```

- [ ] **Step 2: Make it executable**

```bash
chmod +x /Users/danielvaughan/Development/git/codex-resources/scripts/test-sync-to-codex-blog.sh
```

- [ ] **Step 3: Run it to verify it fails**

```bash
cd /Users/danielvaughan/Development/git/codex-resources
bash scripts/test-sync-to-codex-blog.sh
```

Expected: FAIL with `sync script not found or not executable at .../scripts/sync-to-codex-blog.sh`.

This is the expected red state — the sync script doesn't exist yet.

- [ ] **Step 4: Commit**

```bash
cd /Users/danielvaughan/Development/git/codex-resources
git add scripts/test-sync-to-codex-blog.sh
git commit -m "test: add test harness for sync-to-codex-blog script"
```

---

### Task 12: Implement the sync script

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-resources/scripts/sync-to-codex-blog.sh`

- [ ] **Step 1: Write the script**

```bash
#!/usr/bin/env bash
# sync-to-codex-blog.sh
#
# Mirrors articles and sketchnotes from this repo (codex-resources) into a
# checked-out codex-blog working tree. Rewrites sketchnote image paths and
# commits/pushes the result.
#
# Required env:
#   CODEX_BLOG_DIR  Path to checked-out codex-blog working tree.
#   SOURCE_SHA      codex-resources commit SHA used in commit message.
#
# Optional env:
#   SOURCE_DIR      Override source repo root (defaults to git rev-parse).
#   SKIP_PUSH       If set, skip the final `git push` (used by tests).
#
# Exits 0 on success or "nothing to sync". Exits non-zero on any error.

set -euo pipefail

# --- Validate inputs ---
if [ -z "${CODEX_BLOG_DIR:-}" ]; then
  echo "ERROR: CODEX_BLOG_DIR is not set" >&2
  exit 1
fi
if [ ! -d "$CODEX_BLOG_DIR" ]; then
  echo "ERROR: CODEX_BLOG_DIR does not exist: $CODEX_BLOG_DIR" >&2
  exit 1
fi
if [ -z "${SOURCE_SHA:-}" ]; then
  echo "ERROR: SOURCE_SHA is not set" >&2
  exit 1
fi

if [ -z "${SOURCE_DIR:-}" ]; then
  SOURCE_DIR="$(git rev-parse --show-toplevel)"
fi

ARTICLES_SRC="$SOURCE_DIR/articles"
SKETCH_SRC="$SOURCE_DIR/sketchnotes/articles"

if [ ! -d "$ARTICLES_SRC" ]; then
  echo "ERROR: articles source dir missing: $ARTICLES_SRC" >&2
  exit 1
fi
if [ ! -d "$SKETCH_SRC" ]; then
  echo "ERROR: sketchnotes source dir missing: $SKETCH_SRC" >&2
  exit 1
fi

POSTS_DST="$CODEX_BLOG_DIR/_posts"
SKETCH_DST="$CODEX_BLOG_DIR/sketchnotes/articles"
mkdir -p "$POSTS_DST" "$SKETCH_DST"

echo "==> Syncing articles ($ARTICLES_SRC -> $POSTS_DST)"
rsync -av --delete \
  --include='*.md' \
  --exclude='SYNCED-DO-NOT-EDIT.md' \
  --exclude='*' \
  "$ARTICLES_SRC/" "$POSTS_DST/"

echo "==> Syncing sketchnotes ($SKETCH_SRC -> $SKETCH_DST)"
rsync -av --delete \
  --include='*.png' \
  --exclude='SYNCED-DO-NOT-EDIT.md' \
  --exclude='*' \
  "$SKETCH_SRC/" "$SKETCH_DST/"

echo "==> Rewriting sketchnote image paths in synced posts"
# Use sed -i with a portable .bak extension and then delete the .bak files.
# This works on both BSD sed (macOS) and GNU sed (Linux).
find "$POSTS_DST" -name '*.md' -type f \
  -exec sed -i.bak 's|/codex-resources/sketchnotes/|/sketchnotes/|g' {} \;
find "$POSTS_DST" -name '*.bak' -type f -delete

echo "==> Checking for changes in $CODEX_BLOG_DIR"
cd "$CODEX_BLOG_DIR"

if [ -z "$(git status --porcelain)" ]; then
  echo "==> Nothing to sync. Exiting clean."
  exit 0
fi

# Configure bot identity if not already set in env
if [ -z "${GIT_AUTHOR_NAME:-}" ]; then
  git config user.name  "codex-blog-sync[bot]"
  git config user.email "codex-blog-sync@users.noreply.github.com"
fi

SHORT_SHA="${SOURCE_SHA:0:7}"
COMMIT_MSG="chore: sync from codex-resources@${SHORT_SHA}"

git add -A
git commit -m "$COMMIT_MSG"

if [ -z "${SKIP_PUSH:-}" ]; then
  echo "==> Pushing to origin"
  git push
else
  echo "==> SKIP_PUSH set, not pushing"
fi

echo "==> Done."
```

- [ ] **Step 2: Make it executable**

```bash
chmod +x /Users/danielvaughan/Development/git/codex-resources/scripts/sync-to-codex-blog.sh
```

- [ ] **Step 3: Run the test harness**

```bash
cd /Users/danielvaughan/Development/git/codex-resources
bash scripts/test-sync-to-codex-blog.sh
```

Expected: prints `==> Syncing articles ...`, `==> Syncing sketchnotes ...`, `==> Rewriting ...`, `==> Checking for changes ...`, then `PASS: all assertions met`.

If a step fails, fix the script and re-run. Do not proceed until the test passes.

- [ ] **Step 4: Add a "no changes" regression assertion**

Re-running the sync immediately should be a no-op. Add this assertion to the END of `scripts/test-sync-to-codex-blog.sh`, right before the final `echo "PASS: ..."` line:

```bash
# Re-run: should be a clean no-op (script exits 0, no new commit)
COMMIT_BEFORE="$(cd "$DST" && git rev-parse HEAD)"

SKIP_PUSH=1 \
CODEX_BLOG_DIR="$DST" \
SOURCE_SHA="abc1234" \
SOURCE_DIR="$SRC" \
bash "$SYNC_SCRIPT"

COMMIT_AFTER="$(cd "$DST" && git rev-parse HEAD)"
if [ "$COMMIT_BEFORE" != "$COMMIT_AFTER" ]; then
  echo "FAIL: second sync should be a no-op but created a commit" >&2
  exit 1
fi
```

- [ ] **Step 5: Re-run test harness**

```bash
cd /Users/danielvaughan/Development/git/codex-resources
bash scripts/test-sync-to-codex-blog.sh
```

Expected: still passes, with the additional no-op check.

- [ ] **Step 6: Commit**

```bash
cd /Users/danielvaughan/Development/git/codex-resources
git add scripts/sync-to-codex-blog.sh scripts/test-sync-to-codex-blog.sh
git commit -m "feat: add sync-to-codex-blog script and passing test harness"
```

---

### Task 13: Run a real local sync into codex-blog

This is a smoke test against the real codex-resources content and the real codex-blog working tree. The codex-blog repo doesn't have an `origin` remote yet, so use `SKIP_PUSH=1`.

- [ ] **Step 1: Run the sync script against the real repos**

```bash
cd /Users/danielvaughan/Development/git/codex-resources
SKIP_PUSH=1 \
CODEX_BLOG_DIR=/Users/danielvaughan/Development/git/codex-blog \
SOURCE_SHA="$(git rev-parse HEAD)" \
bash scripts/sync-to-codex-blog.sh
```

Expected: lots of `rsync` output, "Rewriting sketchnote image paths", a single commit in codex-blog with message `chore: sync from codex-resources@<sha>`.

- [ ] **Step 2: Verify article count**

```bash
ls /Users/danielvaughan/Development/git/codex-blog/_posts/*.md | wc -l
```

Expected: 164 (or whatever the current count in codex-resources is) — and the marker file `SYNCED-DO-NOT-EDIT.md` is still present.

- [ ] **Step 3: Verify sketchnote count**

```bash
ls /Users/danielvaughan/Development/git/codex-blog/sketchnotes/articles/*.png | wc -l
```

Expected: ~30 (or whatever the current count is).

- [ ] **Step 4: Verify path rewrites**

```bash
grep -rn "/codex-resources/sketchnotes/" /Users/danielvaughan/Development/git/codex-blog/_posts/ || echo "no leftover paths"
```

Expected: `no leftover paths`.

- [ ] **Step 5: Build codex-blog with the synced content**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
rm -rf _site .jekyll-cache
bundle exec jekyll build
```

Expected: build succeeds. Warnings about specific posts are acceptable but read them — front-matter mismatches in any article will surface here.

- [ ] **Step 6: Spot-check a rendered post**

```bash
ls /Users/danielvaughan/Development/git/codex-blog/_site/2026/03/26/ 2>/dev/null | head
```

Expected: at least one post directory exists. Open one of its `index.html` files in a browser and verify the sketchnote `<img>` tag points at `/sketchnotes/articles/...`.

- [ ] **Step 7: Smoke serve**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
bundle exec jekyll serve --no-watch &
SERVE_PID=$!
sleep 5
curl -sf http://localhost:4000/ > /dev/null && echo "home OK"
curl -sf http://localhost:4000/tags/ > /dev/null && echo "tags OK"
curl -sf http://localhost:4000/feed.xml > /dev/null && echo "feed OK"
kill $SERVE_PID
```

Expected: prints "home OK", "tags OK", "feed OK".

- [ ] **Step 8: No new commits in codex-resources**

This step is verification only. The sync committed in codex-blog but not in codex-resources. Don't commit anything in codex-blog yet either — the next phase wires up the action which will commit on its own. If you want to keep the synced state, that's fine; it will be overwritten by the first action run anyway.

---

## Phase 3 — GitHub Action

### Task 14: Write the sync workflow

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-resources/.github/workflows/sync-to-codex-blog.yml`

- [ ] **Step 1: Write the workflow**

```yaml
name: sync-to-codex-blog

on:
  push:
    branches: [main]
    paths:
      - 'articles/**'
      - 'sketchnotes/articles/**'
      - 'scripts/sync-to-codex-blog.sh'
  workflow_dispatch:

concurrency:
  group: sync-to-codex-blog
  cancel-in-progress: false

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout codex-resources
        uses: actions/checkout@v4
        with:
          path: codex-resources

      - name: Checkout codex-blog
        uses: actions/checkout@v4
        with:
          repository: danielvaughan/codex-blog
          token: ${{ secrets.CODEX_BLOG_DEPLOY_TOKEN }}
          path: codex-blog

      - name: Configure git identity
        run: |
          cd codex-blog
          git config user.name  "codex-blog-sync[bot]"
          git config user.email "codex-blog-sync@users.noreply.github.com"

      - name: Run sync script
        env:
          CODEX_BLOG_DIR: ${{ github.workspace }}/codex-blog
          SOURCE_SHA: ${{ github.sha }}
          SOURCE_DIR: ${{ github.workspace }}/codex-resources
        run: |
          bash codex-resources/scripts/sync-to-codex-blog.sh
```

Notes for whoever runs this:
- The `concurrency` block prevents two sync runs from racing each other.
- The `paths` filter only triggers when articles, sketchnotes, or the sync script itself change.
- `CODEX_BLOG_DEPLOY_TOKEN` must be a fine-grained PAT scoped to
  `danielvaughan/codex-blog` with **Contents: Read and write** permission.

- [ ] **Step 2: Verify the YAML is well-formed**

```bash
cd /Users/danielvaughan/Development/git/codex-resources
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/sync-to-codex-blog.yml'))" && echo "YAML OK"
```

Expected: prints `YAML OK`.

If `python3` and `yaml` are unavailable, use any other YAML linter or rely on GitHub to surface errors at run time.

- [ ] **Step 3: Commit**

```bash
cd /Users/danielvaughan/Development/git/codex-resources
git add .github/workflows/sync-to-codex-blog.yml
git commit -m "ci: add sync-to-codex-blog GitHub Action"
```

- [ ] **Step 4: Push codex-resources** (if working from a feature branch, open a PR; if working on main, push directly)

```bash
cd /Users/danielvaughan/Development/git/codex-resources
git push
```

The push will not trigger the sync action yet because `CODEX_BLOG_DEPLOY_TOKEN` is not set. That's expected — Phase 4 sets it up.

---

## Phase 4 — Manual one-time setup (USER ACTIONS)

These cannot be automated. Work through them in order. When done, the auto-sync flow is fully live.

### Task 15 (manual): Create the public repo and push codex-blog

- [ ] **Step 1:** On GitHub, create a new public repository named `codex-blog` under your account (`danielvaughan/codex-blog`). Do NOT initialize with README, .gitignore, or license — the local repo already has these.

- [ ] **Step 2:** Add the remote and push:

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git remote add origin git@github.com:danielvaughan/codex-blog.git
git push -u origin main
```

Expected: branch `main` pushed to `origin/main`.

---

### Task 16 (manual): Configure GitHub Pages and DNS

- [ ] **Step 1:** In the codex-blog repo on GitHub: **Settings → Pages**.
  - **Source:** Deploy from a branch
  - **Branch:** `main`, folder `/`
  - **Custom domain:** `codex.danielvaughan.com`

  GitHub will save the CNAME file (already in repo) and start checking DNS.

- [ ] **Step 2:** At your DNS provider, add a `CNAME` record:
  - Name: `codex`
  - Value: `danielvaughan.github.io.` (note the trailing dot)
  - TTL: default

- [ ] **Step 3:** Wait for DNS propagation (minutes to hours). Verify with:

```bash
dig +short codex.danielvaughan.com CNAME
```

Expected: prints `danielvaughan.github.io.` once propagated.

- [ ] **Step 4:** In **Settings → Pages**, enable **Enforce HTTPS** once GitHub stops showing a "DNS check in progress" warning. This may take up to 24 hours after DNS resolves.

- [ ] **Step 5:** Visit `https://codex.danielvaughan.com` in a browser. Expected: home page renders. Posts list will be empty until the first sync.

---

### Task 17 (manual): Create the deploy PAT and store it as a secret

- [ ] **Step 1:** GitHub → your profile → **Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**.
  - **Token name:** `codex-blog-sync`
  - **Expiration:** your call (90 days, 1 year, or no expiration)
  - **Repository access:** Only select repositories → `danielvaughan/codex-blog`
  - **Permissions:** Repository permissions → **Contents: Read and write**

- [ ] **Step 2:** Copy the token immediately (it is shown only once).

- [ ] **Step 3:** In the **codex-resources** repo on GitHub: **Settings → Secrets and variables → Actions → New repository secret**.
  - **Name:** `CODEX_BLOG_DEPLOY_TOKEN`
  - **Value:** paste the token

---

### Task 18 (manual): First sync and end-to-end smoke test

- [ ] **Step 1:** In the codex-resources repo on GitHub: **Actions → sync-to-codex-blog → Run workflow** (on `main`).

- [ ] **Step 2:** Watch the run. Expected: green check. The sync script logs each step.

- [ ] **Step 3:** Visit the codex-blog repo on GitHub. Expected: a new commit `chore: sync from codex-resources@<sha>` from `codex-blog-sync[bot]`. `_posts/` and `sketchnotes/articles/` should now be populated.

- [ ] **Step 4:** Watch GitHub Pages rebuild. **Actions** tab in codex-blog → **pages-build-deployment** → green check.

- [ ] **Step 5:** Visit `https://codex.danielvaughan.com/`. Expected: home page lists posts in reverse-chrono order. Click into one. Expected: the sketchnote `<img>` loads from `https://codex.danielvaughan.com/sketchnotes/articles/...`.

- [ ] **Step 6:** Visit `https://codex.danielvaughan.com/tags/`. Expected: tag index renders with all tags from the synced posts.

- [ ] **Step 7:** Visit `https://codex.danielvaughan.com/feed.xml`. Expected: valid Atom feed with the posts.

- [ ] **Step 8:** Edit a single article in codex-resources, push to main. Expected: sync workflow triggers automatically, codex-blog gets a new commit, Pages rebuilds, the change appears live within a couple of minutes.

If any step fails: check the workflow run logs. The most common failures are:
- PAT misconfigured or expired → workflow's checkout step fails with 403/401.
- DNS not propagated → custom domain shows a 404 or "improperly configured" warning.
- Front-matter parse error in a specific article → Pages build log will name the file.

---

## Out-of-scope items intentionally not in this plan

- Search-engine sitemap submission, analytics, social-card images.
- Per-tag archive URLs (`/tags/codex-cli/`). Single tag-index page is enough for now.
- Comments / Disqus / Giscus.
- Any kind of test in codex-blog itself — Jekyll's own build is the test.
- Two-way sync, draft handling, scheduled publishing.

If you want any of these later, they slot in as additive changes to `_config.yml` and `_pages/` without touching the sync flow.