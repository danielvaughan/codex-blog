# codex-blog ↔ main site code-sharing refactor — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace minimal-mistakes in codex-blog with a Jekyll site that imports SCSS/layouts/includes from a git submodule of `danielvaughan.github.io`, so the blog literally shares source code with the main site.

**Architecture:** Submodule `danielvaughan.github.io` at `_main-site/`. Symlink three subdirectories (`_includes/main-site`, `_layouts/main-site`, `_sass/main-site`) into the submodule. `assets/css/main.scss` imports the main site's full SCSS pipeline plus a slim codex-blog override layer. Custom layouts (`default.html`, `home.html`, `post.html`, `page.html`) use `{% include main-site/head.html %}`. The sticky navy navbar and one-line footer remain codex-blog-specific includes.

**Tech Stack:** Jekyll 3.x via `github-pages` gem, Bootstrap 4 + Bootswatch Minty + Montserrat (inherited from main site), git submodules, SCSS, kramdown, jekyll-paginate v1, jekyll-feed.

**Spec:** [`docs/superpowers/specs/2026-04-07-share-code-with-main-site-design.md`](../specs/2026-04-07-share-code-with-main-site-design.md)

**Reference source:** `/Users/danielvaughan/Development/git/danielvaughan.github.io/`

---

## Pre-flight

Both repos must be in clean state before starting. The work happens in:
- `/Users/danielvaughan/Development/git/codex-blog` (codex-blog itself, on `main`)
- `/Users/danielvaughan/Development/git/codex-resources-blog-sync` (sync script worktree, on `add-codex-blog-sync`)

```bash
git -C /Users/danielvaughan/Development/git/codex-blog status
git -C /Users/danielvaughan/Development/git/codex-resources-blog-sync status
```
Expected: both `nothing to commit, working tree clean`. If not, stop and report BLOCKED.

```bash
git -C /Users/danielvaughan/Development/git/codex-blog log --oneline | head -3
```
Should show the recent layout/footer fix commits at the top, including `236d04f fix(layout): reclaim wasted left margin, fix teal footer band`.

```bash
ls /Users/danielvaughan/Development/git/danielvaughan.github.io/_sass/_variables.scss
```
Expected: file exists. The main site source must be present for the submodule add to work locally.

---

## File Structure

### Files created in codex-blog

| Path | Purpose |
|---|---|
| `.gitmodules` | Tracks the `_main-site` submodule URL and path |
| `_main-site/` | Git submodule (read-only) → `danielvaughan.github.io` |
| `_includes/main-site` | Symlink → `../_main-site/_includes/` |
| `_layouts/main-site` | Symlink → `../_main-site/_layouts/` |
| `_sass/main-site` | Symlink → `../_main-site/_sass/` |
| `_includes/dv-navbar.html` | Sticky navy navbar (Bootstrap 4 markup) |
| `_includes/dv-footer.html` | One-line copyright footer |
| `_layouts/default.html` | Wraps body in `main-site/head.html` + dv-navbar + main + dv-footer |
| `_layouts/post.html` | Article: header + tag badges + inline TOC + body |
| `_layouts/home.html` | Paginated card list + intro band |
| `_layouts/page.html` | Static page wrapper for `/tags/` |
| `_sass/codex-blog.scss` | Slim codex-blog override layer (~150 lines) |

### Files modified in codex-blog

| Path | What changes |
|---|---|
| `assets/css/main.scss` | Replace minimal-mistakes imports with main-site Bootstrap stack + codex-blog override |
| `_includes/head/custom.html` | Drop a11y JS shims, add inline TOC builder JS, keep Montserrat link + mermaid |
| `_config.yml` | Drop `remote_theme`, drop search, add layout defaults, exclude `_main-site`, add `sass.load_paths` |
| `Gemfile` | Drop `jekyll-remote-theme` |

### Files deleted in codex-blog

| Path | Why |
|---|---|
| `_includes/masthead.html` | Replaced by `dv-navbar.html` |
| `_includes/footer.html` | Replaced by `dv-footer.html` |

### Files modified in codex-resources worktree

| Path | What changes |
|---|---|
| `scripts/sync-to-codex-blog.sh` | Layout normalisation now strips `^layout: default$` and `^layout: single$` lines entirely (was: rewrite to `single`) |
| `scripts/test-sync-to-codex-blog.sh` | Test fixtures + assertions updated for the strip-not-rewrite behaviour |

### Files NOT touched

- `_posts/*.md` and `sketchnotes/articles/*` are auto-synced; we apply a one-shot rescue commit in step 13 to strip layout lines locally so the new defaults take effect immediately, but the source articles in codex-resources stay untouched.
- `CNAME`, `README.md`, `.gitignore`, `index.html`, `_pages/tags.md`, `assets/images/daniel-vaughan-avatar.jpg` remain unchanged.

---

## Task 1: Add the main site as a git submodule

**Files:**
- Create: `.gitmodules`
- Create: `_main-site/` (git submodule entry, not a regular directory)

- [ ] **Step 1: Add the submodule from the GitHub HTTPS clone URL**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git submodule add https://github.com/danielvaughan/danielvaughan.github.io.git _main-site
```
Expected: clones `danielvaughan.github.io` into `_main-site/` and creates `.gitmodules` with the URL and path.

- [ ] **Step 2: Verify submodule contents**

```bash
ls /Users/danielvaughan/Development/git/codex-blog/_main-site/_sass/_variables.scss
ls /Users/danielvaughan/Development/git/codex-blog/_main-site/_includes/head.html
ls /Users/danielvaughan/Development/git/codex-blog/_main-site/_layouts/default.html
```
Expected: all three files exist. If any missing, stop and report BLOCKED — the submodule clone failed or the main site repo's structure has changed.

- [ ] **Step 3: Inspect .gitmodules**

```bash
cat /Users/danielvaughan/Development/git/codex-blog/.gitmodules
```
Expected output:
```
[submodule "_main-site"]
	path = _main-site
	url = https://github.com/danielvaughan/danielvaughan.github.io.git
```

- [ ] **Step 4: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add .gitmodules _main-site
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "chore: add danielvaughan.github.io as _main-site submodule"
```

---

## Task 2: Create the symlinks

**Files:**
- Create: `_includes/main-site` (symlink to `../_main-site/_includes`)
- Create: `_layouts/main-site` (symlink to `../_main-site/_layouts`)
- Create: `_sass/main-site` (symlink to `../_main-site/_sass`)

These symlinks let Jekyll resolve `{% include main-site/head.html %}` and `@import "main-site/variables"` against the submodule contents.

- [ ] **Step 1: Create the symlinks**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
mkdir -p _includes _layouts _sass
ln -s ../_main-site/_includes _includes/main-site
ln -s ../_main-site/_layouts  _layouts/main-site
ln -s ../_main-site/_sass     _sass/main-site
```

- [ ] **Step 2: Verify the symlinks resolve**

```bash
ls -l /Users/danielvaughan/Development/git/codex-blog/_includes/main-site
ls -l /Users/danielvaughan/Development/git/codex-blog/_layouts/main-site
ls -l /Users/danielvaughan/Development/git/codex-blog/_sass/main-site
ls /Users/danielvaughan/Development/git/codex-blog/_includes/main-site/head.html
ls /Users/danielvaughan/Development/git/codex-blog/_sass/main-site/_variables.scss
```
Expected: each `ls -l` shows `-> ../_main-site/_<dir>` and the file lookups succeed.

- [ ] **Step 3: Commit (symlinks are tracked by git as `mode 120000`)**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add _includes/main-site _layouts/main-site _sass/main-site
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "chore: add symlinks into _main-site for SCSS/includes/layouts"
```

---

## Task 3: Update Gemfile (drop jekyll-remote-theme)

**Files:**
- Modify: `/Users/danielvaughan/Development/git/codex-blog/Gemfile`

- [ ] **Step 1: Read existing Gemfile**

```bash
cat /Users/danielvaughan/Development/git/codex-blog/Gemfile
```
Expected: contains `gem "github-pages"`, `gem "webrick"`, and a `jekyll_plugins` group with `jekyll-remote-theme`. If different, stop and report.

- [ ] **Step 2: Replace the file**

Overwrite `/Users/danielvaughan/Development/git/codex-blog/Gemfile` with EXACTLY:

```ruby
source "https://rubygems.org"

# GitHub Pages — pinned gem set used by github.io builds
gem "github-pages", group: :jekyll_plugins

# Required for `bundle exec jekyll serve` on Ruby 3+
gem "webrick"

# Plugins explicitly used in _config.yml
group :jekyll_plugins do
  gem "jekyll-seo-tag"
  gem "jekyll-sitemap"
  gem "jekyll-feed"
  gem "jekyll-paginate"
  gem "jekyll-include-cache"
end
```

The diff: `jekyll-remote-theme` line is gone.

- [ ] **Step 3: Run bundle install**

```bash
cd /Users/danielvaughan/Development/git/codex-blog && bundle install
```
Expected: `Bundle complete!`. The github-pages gem still pulls in the same dependency tree minus the remote-theme integration.

- [ ] **Step 4: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add Gemfile
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "chore: drop jekyll-remote-theme from Gemfile"
```

---

## Task 4: Replace assets/css/main.scss with the new import chain

**Files:**
- Modify: `/Users/danielvaughan/Development/git/codex-blog/assets/css/main.scss`

This task replaces the file completely. The new file is small — just imports.

- [ ] **Step 1: Read existing main.scss to confirm size and structure before replacing**

```bash
wc -l /Users/danielvaughan/Development/git/codex-blog/assets/css/main.scss
head -5 /Users/danielvaughan/Development/git/codex-blog/assets/css/main.scss
```
Expected: ~600 lines (the current MM override block). The first lines should be `---` front matter and `@charset`.

- [ ] **Step 2: Overwrite with the new file**

Replace `/Users/danielvaughan/Development/git/codex-blog/assets/css/main.scss` entirely with EXACTLY this content:

```scss
---
# Front matter required so Jekyll processes this file as SCSS
---

@charset "utf-8";

// Pull in the main site's full SCSS pipeline via the _main-site submodule.
// These five imports are the EXACT five lines danielvaughan.github.io
// uses in its own assets/main.scss — codex-blog inherits everything for
// free: colours, Montserrat headings, .profile-nav, .content-section,
// .book-card, .connect-btn, the entire visual language.
@import "main-site/variables";
@import "main-site/bootstrap/bootstrap";
@import "main-site/syntax-highlighting";
@import "main-site/bootstrap-4-jekyll/bootstrap-4-jekyll";
@import "main-site/bootstrap_customization";

// codex-blog overrides on top of the main site's stack
@import "codex-blog";
```

- [ ] **Step 3: Note that this build will fail until Task 5 creates `_sass/codex-blog.scss`** — that's expected. We'll verify the build at the end of Task 5, not here.

- [ ] **Step 4: Commit (so the failing intermediate state is captured cleanly before the next file is added)**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add assets/css/main.scss
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "refactor(scss): replace MM imports with main-site Bootstrap pipeline"
```

---

## Task 5: Create the codex-blog SCSS override layer

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/_sass/codex-blog.scss`

- [ ] **Step 1: Write the file**

Create `/Users/danielvaughan/Development/git/codex-blog/_sass/codex-blog.scss` with EXACTLY this content:

```scss
// =============================================================================
// codex-blog overrides on top of the main site's Bootstrap stack
// =============================================================================
// All variables ($primary, $body-color, $gray-100..900, $headings-font-family,
// $border-radius, etc.) come from main-site/_variables.scss which is imported
// before this file in assets/css/main.scss.

// ---------- Sticky navy navbar (matches main site .profile-nav exactly) ----
// The navbar markup uses `class="profile-nav navbar navbar-expand-lg"` so it
// inherits the .profile-nav rules from the main site's
// _bootstrap_customization.scss for free. We only add styles for the avatar
// image (which the main site doesn't have).
.profile-nav {
  .navbar-brand {
    display: inline-flex;
    align-items: center;
  }
  .navbar-avatar {
    width: 28px;
    height: 28px;
    border-radius: 50%;
    border: 2px solid rgba(255, 255, 255, 0.35);
    object-fit: cover;
    margin-right: 0.5rem;
  }
}

// ---------- Footer top border colour ----------
// .py-5 .border-top from Bootstrap gives the structure; we just tighten the
// border colour to match the main site's #eceeef instead of Bootstrap default.
.dv-footer {
  border-top-color: #eceeef !important;
}

// ---------- Post header ----------
.post-header {
  margin-bottom: 2rem;
}
.post-title {
  font-family: $headings-font-family;
  font-weight: 700;
  color: $gray-800;
  letter-spacing: -0.015em;
  margin-bottom: 0.5rem;
}
.post-meta {
  font-size: 0.85rem;
  margin-bottom: 0.5rem;
}

// ---------- Tag badges (used on post header and home page cards) ----------
.post-tags {
  margin-top: 0.75rem;
}
.post-tags .badge {
  display: inline-block;
  margin: 0 0.25rem 0.25rem 0;
  padding: 0.3em 0.7em;
  font-family: $headings-font-family;
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  color: $primary;
  background: #f0f4ff;
  border: 1px solid #d9e2ff;
  border-radius: 999px;
  text-decoration: none;
  &:hover {
    background: $primary;
    color: #fff;
    text-decoration: none;
  }
}

// ---------- Inline TOC at top of articles ----------
// Populated by JS in _includes/head/custom.html.
.post-toc {
  background: $gray-100;
  border-left: 4px solid $primary;
  padding: 1rem 1.25rem;
  margin: 1.5rem 0 2rem;
}
.post-toc-title {
  font-family: $headings-font-family;
  font-size: 0.85rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  color: $gray-800;
  margin-bottom: 0.4rem;
}
.post-toc ul {
  margin: 0;
  padding-left: 1.25rem;
  list-style: disc;
}
.post-toc li {
  font-size: 0.95rem;
  line-height: 1.5;
  a {
    color: $primary;
    text-decoration: none;
    &:hover { text-decoration: underline; }
  }
}
.post-toc[hidden] {
  display: none;
}

// ---------- Post body content ----------
// Sketchnote at top of articles (when first child of body is a paragraph
// containing only one image) — clean treatment with shadow.
.post-body > p:first-child > img:only-child {
  display: block;
  margin: 0 auto 2rem;
  max-width: 100%;
  height: auto;
  border-radius: 4px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.08);
}
.post-body img {
  max-width: 100%;
  height: auto;
}
// Better h2 / h3 hierarchy in body (the main site's content-section h2 has the
// blue underline accent — extend that to in-body h2 inside .post-body).
.post-body h2 {
  font-size: 1.55rem;
  font-weight: 700;
  color: $gray-800;
  margin-top: 2.5rem;
  margin-bottom: 1rem;
  padding-bottom: 0.6rem;
  border-bottom: 3px solid $primary;
  display: inline-block;
}
.post-body h3 {
  font-size: 1rem;
  font-weight: 700;
  color: $primary;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  margin-top: 2rem;
  margin-bottom: 0.5rem;
}
.post-body h4 {
  font-size: 1.05rem;
  font-weight: 700;
  margin-top: 1.5rem;
}
.post-body table {
  display: block;
  overflow-x: auto;
  width: 100%;
  font-size: 0.92rem;
  margin: 1.5rem 0;
}
.post-body blockquote {
  border-left: 4px solid $primary;
  background: $gray-100;
  padding: 0.75rem 1.25rem;
  margin: 1.5rem 0;
  color: #4a4a4a;
  p:last-child { margin-bottom: 0; }
}
.post-body pre {
  font-size: 14px;
  line-height: 1.5;
  border-radius: 6px;
}
// Inline code only (not inside <pre>): give it the chip background
.post-body :not(pre) > code {
  background: #f0f4f8;
  color: #0d1b4b;
  padding: 0.1em 0.35em;
  border-radius: 3px;
  font-size: 0.88em;
}

// ---------- Home page card list ----------
// .book-card is defined in main-site/_bootstrap_customization.scss with a
// left blue border, white background, and padding. We add a few overrides
// for codex-blog post cards (longer titles, excerpt styling).
.post-card {
  margin-bottom: 1.5rem;
  .book-title {
    font-size: 1.15rem;
    font-weight: 700;
    line-height: 1.3;
    margin-bottom: 0.3rem;
    a {
      color: $gray-800;
      text-decoration: none;
      &:hover { color: $primary; text-decoration: none; }
    }
  }
  .book-meta {
    font-size: 0.85rem;
    color: $gray-600;
    margin-bottom: 0.4rem;
  }
  .post-excerpt {
    color: $gray-700;
    font-size: 0.95rem;
    line-height: 1.6;
    margin-top: 0.5rem;
    margin-bottom: 0;
  }
}

// ---------- Pagination ----------
.pagination {
  margin-top: 2rem;
  .page-link {
    color: $primary;
  }
  .page-item.disabled .page-link {
    color: $gray-600;
  }
}

// ---------- Skip-to-content link (a11y) ----------
.sr-only {
  position: absolute !important;
  width: 1px !important;
  height: 1px !important;
  padding: 0 !important;
  margin: -1px !important;
  overflow: hidden !important;
  clip: rect(0,0,0,0) !important;
  white-space: nowrap !important;
  border: 0 !important;
}
.sr-only-focusable:focus,
.sr-only-focusable:focus-visible {
  position: fixed !important;
  top: 8px !important;
  left: 8px !important;
  width: auto !important;
  height: auto !important;
  padding: 10px 18px !important;
  margin: 0 !important;
  overflow: visible !important;
  clip: auto !important;
  background: #0d1b4b !important;
  color: #fff !important;
  font-family: $headings-font-family !important;
  font-size: 14px !important;
  font-weight: 600 !important;
  text-decoration: none !important;
  border-radius: 4px !important;
  z-index: 9999 !important;
}

// ---------- Focus-visible rings (a11y) ----------
:focus-visible {
  outline: 2px solid $primary;
  outline-offset: 2px;
}
.profile-nav :focus-visible {
  outline-color: #fff;
  outline-offset: 3px;
}

// ---------- Reduced motion ----------
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

- [ ] **Step 2: Note this build will still fail** because Task 6+ haven't created the layouts yet. We just verify the SCSS compiles in isolation. Run:

```bash
cd /Users/danielvaughan/Development/git/codex-blog && bundle exec jekyll build 2>&1 | tail -15
```
Expected: SCSS compiles (no `Conversion error: Jekyll::Converters::Scss`). It WILL fail at layout resolution because `_layouts/post.html` etc. don't exist yet, but the SCSS section of the build should pass.

If the SCSS errors out, the most likely cause is a missing variable or import. Read the error line, fix `_sass/codex-blog.scss`, retry.

- [ ] **Step 3: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add _sass/codex-blog.scss
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "feat(scss): add codex-blog override layer"
```

---

## Task 6: Create _layouts/default.html

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/_layouts/default.html`

- [ ] **Step 1: Write the file**

Create with EXACTLY this content:

```html
<!DOCTYPE html>
<html lang="{{ page.lang | default: site.lang | default: "en" }}">
  {%- include main-site/head.html -%}
  <body>
    {%- include dv-navbar.html -%}
    <main id="main" role="main" class="py-5">
      <div class="container page-content">
        {{ content }}
      </div>
    </main>
    {%- include dv-footer.html -%}
  </body>
</html>
```

- [ ] **Step 2: Note that the include of `main-site/head.html` is the symlink path** — Jekyll's include resolution looks at `_includes/main-site/head.html` which is the symlinked file inside the submodule.

- [ ] **Step 3: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add _layouts/default.html
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "feat(layout): add codex-blog default.html using main-site head"
```

---

## Task 7: Create _layouts/post.html

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/_layouts/post.html`

- [ ] **Step 1: Write the file**

Create with EXACTLY this content:

```html
---
layout: default
---
<article class="post">
  <header class="post-header">
    <h1 class="post-title">{{ page.title | escape }}</h1>
    <p class="post-meta text-muted">
      <time datetime="{{ page.date | date_to_xmlschema }}">
        {{ page.date | date: "%B %-d, %Y" }}
      </time>
    </p>

    {%- if page.tags and page.tags.size > 0 -%}
    <div class="post-tags">
      {%- for tag in page.tags -%}
      <a href="{{ '/tags/#' | append: tag | slugify | relative_url }}" class="badge">{{ tag }}</a>
      {%- endfor -%}
    </div>
    {%- endif -%}
  </header>

  <nav class="post-toc" aria-label="On this page" hidden>
    <p class="post-toc-title">On this page</p>
    <ul></ul>
  </nav>

  <section class="post-body">
    {{ content }}
  </section>
</article>
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add _layouts/post.html
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "feat(layout): add post.html with header, tags, inline TOC, body"
```

---

## Task 8: Create _layouts/home.html

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/_layouts/home.html`

- [ ] **Step 1: Write the file**

Create with EXACTLY this content:

```html
---
layout: default
---
<section class="content-section bg-alt">
  <div class="container">
    <h2>About this blog</h2>
    <p>Notes on agentic software engineering, Codex CLI, and how AI is changing the way we build software. Published as I write them.</p>
  </div>
</section>

<section class="content-section bg-white">
  <div class="container">
    {%- for post in paginator.posts -%}
    <article class="book-card post-card">
      <div class="book-title">
        <a href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
      </div>
      <div class="book-meta">
        <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%B %-d, %Y" }}</time>
      </div>
      {%- if post.tags and post.tags.size > 0 -%}
      <div class="post-tags">
        {%- for tag in post.tags limit:5 -%}
        <a href="{{ '/tags/#' | append: tag | slugify | relative_url }}" class="badge">{{ tag }}</a>
        {%- endfor -%}
      </div>
      {%- endif -%}
      <p class="post-excerpt">{{ post.excerpt | strip_html | truncatewords: 30 }}</p>
    </article>
    {%- endfor -%}

    {%- if paginator.total_pages > 1 -%}
    <nav aria-label="Pagination" class="mt-4">
      <ul class="pagination">
        {%- if paginator.previous_page -%}
        <li class="page-item">
          <a class="page-link" href="{{ paginator.previous_page_path | relative_url }}">← Newer</a>
        </li>
        {%- endif -%}
        <li class="page-item disabled"><span class="page-link">Page {{ paginator.page }} of {{ paginator.total_pages }}</span></li>
        {%- if paginator.next_page -%}
        <li class="page-item">
          <a class="page-link" href="{{ paginator.next_page_path | relative_url }}">Older →</a>
        </li>
        {%- endif -%}
      </ul>
    </nav>
    {%- endif -%}
  </div>
</section>
```

- [ ] **Step 2: Note** — `index.html` already has front matter `layout: home`, so once this file exists Jekyll will use it. Verified separately later.

- [ ] **Step 3: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add _layouts/home.html
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "feat(layout): add home.html with paginated card list and intro band"
```

---

## Task 9: Create _layouts/page.html

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/_layouts/page.html`

- [ ] **Step 1: Write the file**

Create with EXACTLY this content:

```html
---
layout: default
---
<article class="page">
  {%- if page.title -%}
  <header class="post-header">
    <h1 class="post-title">{{ page.title | escape }}</h1>
  </header>
  {%- endif -%}
  <section class="post-body">
    {{ content }}
  </section>
</article>
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add _layouts/page.html
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "feat(layout): add page.html for static pages like /tags/"
```

---

## Task 10: Create _includes/dv-navbar.html

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/_includes/dv-navbar.html`

- [ ] **Step 1: Write the file**

Create with EXACTLY this content:

```html
<a class="sr-only sr-only-focusable" href="#main">Skip to content</a>
<nav id="site-nav" class="profile-nav navbar navbar-expand-lg" aria-label="Primary">
  <div class="container">
    <a class="navbar-brand" href="{{ '/' | relative_url }}">
      <img src="{{ '/assets/images/daniel-vaughan-avatar.jpg' | relative_url }}"
           alt="" width="28" height="28" class="navbar-avatar">
      <span>Daniel Vaughan</span>
    </a>
    <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#dvNavCollapse"
            aria-controls="dvNavCollapse" aria-expanded="false" aria-label="Toggle navigation">
      <span class="navbar-toggler-icon"></span>
    </button>
    <div class="collapse navbar-collapse" id="dvNavCollapse">
      <ul class="navbar-nav ml-auto">
        <li class="nav-item"><a class="nav-link" href="{{ '/' | relative_url }}">Home</a></li>
        <li class="nav-item"><a class="nav-link" href="{{ '/tags/' | relative_url }}">Tags</a></li>
        <li class="nav-item"><a class="nav-link" href="https://danielvaughan.com/" rel="noopener">Main site →</a></li>
      </ul>
    </div>
  </div>
</nav>
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add _includes/dv-navbar.html
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "feat(include): add dv-navbar.html (uses .profile-nav from main site)"
```

---

## Task 11: Create _includes/dv-footer.html

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/_includes/dv-footer.html`

- [ ] **Step 1: Write the file**

Create with EXACTLY this content:

```html
<footer class="dv-footer py-5 border-top">
  <div class="container">
    <p>Copyright © 2000–{{ 'now' | date: "%Y" }}. Daniel Vaughan. All rights reserved</p>
  </div>
</footer>
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add _includes/dv-footer.html
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "feat(include): add dv-footer.html (Bootstrap py-5 border-top, container)"
```

---

## Task 12: Update _includes/head/custom.html

**Files:**
- Modify: `/Users/danielvaughan/Development/git/codex-blog/_includes/head/custom.html`

- [ ] **Step 1: Read existing file**

```bash
cat /Users/danielvaughan/Development/git/codex-blog/_includes/head/custom.html
```
Expected: contains the Montserrat link, an inline a11y JS script, and the mermaid `<script type="module">`. If different, stop and report.

- [ ] **Step 2: Replace with new content**

Overwrite `/Users/danielvaughan/Development/git/codex-blog/_includes/head/custom.html` with EXACTLY:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Montserrat:wght@400;600;700&display=swap" rel="stylesheet">

<script>
  // Inline TOC builder — populates nav.post-toc from .post-body h2 elements
  document.addEventListener('DOMContentLoaded', function () {
    var nav = document.querySelector('nav.post-toc');
    if (!nav) return;
    var ul = nav.querySelector('ul');
    var headings = document.querySelectorAll('.post-body h2');
    if (headings.length === 0) {
      nav.setAttribute('hidden', '');
      return;
    }
    headings.forEach(function (h) {
      var id = h.id || h.textContent
        .toLowerCase()
        .replace(/[^\w\s-]/g, '')
        .trim()
        .replace(/\s+/g, '-')
        .replace(/^-|-$/g, '');
      h.id = id;
      var li = document.createElement('li');
      var a = document.createElement('a');
      a.href = '#' + id;
      a.textContent = h.textContent;
      li.appendChild(a);
      ul.appendChild(li);
    });
    nav.removeAttribute('hidden');
  });
</script>

{% if site.mermaid %}
<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@{{ site.mermaid.version | default: 10 }}/dist/mermaid.esm.min.mjs';

  document.addEventListener('DOMContentLoaded', () => {
    // kramdown emits ```mermaid fences as <pre><code class="language-mermaid">...
    // mermaid.js expects <div class="mermaid">...</div>. Convert in place.
    document.querySelectorAll('pre > code.language-mermaid').forEach((code) => {
      const div = document.createElement('div');
      div.className = 'mermaid';
      div.textContent = code.textContent;
      code.parentNode.parentNode.replaceChild(div, code.parentNode);
    });

    mermaid.initialize({ startOnLoad: false, theme: 'default', securityLevel: 'loose' });
    mermaid.run();
  });
</script>
{% endif %}
```

The diff: dropped the a11y JS shims that added `role="main"` to `#main` and `aria-label` to TOC/pagination navs (no longer needed because the new layouts use proper `<main>` and explicit `aria-label` natively). Added the inline TOC builder JS.

- [ ] **Step 3: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add _includes/head/custom.html
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "refactor(head): drop a11y JS shims, add inline TOC builder"
```

---

## Task 13: Update _config.yml

**Files:**
- Modify: `/Users/danielvaughan/Development/git/codex-blog/_config.yml`

- [ ] **Step 1: Read existing file**

```bash
cat /Users/danielvaughan/Development/git/codex-blog/_config.yml
```
Expected: has `remote_theme:`, `minimal_mistakes_skin:`, `search:`, `jekyll-remote-theme` plugin, layout default `single`, etc. If different, stop and report.

- [ ] **Step 2: Replace with new content**

Overwrite `/Users/danielvaughan/Development/git/codex-blog/_config.yml` with EXACTLY:

```yaml
# Site
title: "Codex Blog"
description: "Articles on agentic software engineering with Codex CLI"
url: "https://codex.danielvaughan.com"
baseurl: ""
repository: "danielvaughan/codex-blog"

# Build
permalink: /:year/:month/:day/:title/
timezone: Europe/London
markdown: kramdown

# Pagination
paginate: 10
paginate_path: "/page:num/"

# Mermaid (rendered via _includes/head/custom.html)
mermaid:
  version: "10"

# Plugins (jekyll-remote-theme is gone — minimal-mistakes is gone)
plugins:
  - jekyll-seo-tag
  - jekyll-sitemap
  - jekyll-feed
  - jekyll-paginate
  - jekyll-include-cache

# SCSS load paths so @import "main-site/..." resolves via the symlink
sass:
  load_paths:
    - _sass

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
  - scope:
      path: ""
      type: posts
    values:
      layout: post
  - scope:
      path: "_pages"
      type: pages
    values:
      layout: page

# Include _pages so Jekyll picks them up
include:
  - _pages

# Exclude
exclude:
  - _main-site
  - Gemfile
  - Gemfile.lock
  - vendor/
  - .bundle/
  - docs/
  - README.md
  - "*.gem"
  - "*.gemspec"
  - scripts/
```

- [ ] **Step 3: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add _config.yml
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "refactor(config): drop minimal-mistakes, add layout defaults, exclude _main-site"
```

---

## Task 14: Delete obsolete includes

**Files:**
- Delete: `/Users/danielvaughan/Development/git/codex-blog/_includes/masthead.html`
- Delete: `/Users/danielvaughan/Development/git/codex-blog/_includes/footer.html`

These are replaced by `dv-navbar.html` and `dv-footer.html`. Nothing references them anymore.

- [ ] **Step 1: Verify they exist**

```bash
ls /Users/danielvaughan/Development/git/codex-blog/_includes/masthead.html
ls /Users/danielvaughan/Development/git/codex-blog/_includes/footer.html
```
Expected: both files exist.

- [ ] **Step 2: Delete with git rm**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog rm _includes/masthead.html _includes/footer.html
```
Expected: prints `rm '_includes/masthead.html'` and `rm '_includes/footer.html'`.

- [ ] **Step 3: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "chore: remove obsolete masthead.html and footer.html includes"
```

---

## Task 15: Apply the local layout-strip rescue to existing _posts

**Files:**
- Modify: every `/Users/danielvaughan/Development/git/codex-blog/_posts/*.md` that has a `layout:` line set to `single` or `default`

The new `_config.yml` defaults set `layout: post` but per-post front matter overrides defaults. The synced posts currently have `layout: single` from the previous sync script normalisation. Strip those lines locally so the default takes effect.

- [ ] **Step 1: Count posts that have a `layout: single` or `layout: default` line**

```bash
/usr/bin/grep -lE '^layout: (single|default)$' /Users/danielvaughan/Development/git/codex-blog/_posts/*.md | wc -l
```
Expected: a positive number (likely 100+).

- [ ] **Step 2: Apply the strip**

```bash
find /Users/danielvaughan/Development/git/codex-blog/_posts -name '*.md' -type f \
  -exec /usr/bin/sed -i.bak \
    -e '/^layout: single$/d' \
    -e '/^layout: default$/d' \
    {} \;
find /Users/danielvaughan/Development/git/codex-blog/_posts -name '*.bak' -delete
```

- [ ] **Step 3: Verify zero remaining `layout:` lines in _posts (other than the marker file's own front matter)**

```bash
/usr/bin/grep -lE '^layout: (single|default)$' /Users/danielvaughan/Development/git/codex-blog/_posts/*.md | wc -l
```
Expected: `0`.

- [ ] **Step 4: Verify build works end-to-end**

```bash
cd /Users/danielvaughan/Development/git/codex-blog && rm -rf _site .jekyll-cache && bundle exec jekyll build 2>&1 | tail -10
```
Expected: `done in N.NNN seconds`. Pre-existing Liquid template warnings on the 7 articles with `{{ ... }}` in code blocks are expected.

- [ ] **Step 5: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add _posts/
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "fix(rescue): strip layout: single/default from synced posts

The new _config.yml default 'layout: post' can't override per-post front
matter, so the synced posts (which have 'layout: single' from the previous
sync script normalisation) need their layout line stripped locally.

Once the codex-resources sync script update lands and the action re-runs,
the bot's sync commit will produce identical content."
```

---

## Task 16: Local visual smoke test

This task is verification only — no commits.

- [ ] **Step 1: Stop any existing local server**

```bash
pkill -f "jekyll serve" 2>/dev/null || true
sleep 1
```

- [ ] **Step 2: Clean rebuild**

```bash
cd /Users/danielvaughan/Development/git/codex-blog && rm -rf _site .jekyll-cache && bundle exec jekyll build 2>&1 | tail -5
```
Expected: build completes. The liquid warnings on 7 articles are still expected. No SCSS errors. No Liquid render errors.

- [ ] **Step 3: Verify expected output files**

```bash
test -f /Users/danielvaughan/Development/git/codex-blog/_site/index.html && echo "index OK"
test -f /Users/danielvaughan/Development/git/codex-blog/_site/tags/index.html && echo "tags OK"
test -f /Users/danielvaughan/Development/git/codex-blog/_site/CNAME && echo "CNAME OK"
test -f /Users/danielvaughan/Development/git/codex-blog/_site/assets/css/main.css && echo "main.css OK"
test -f /Users/danielvaughan/Development/git/codex-blog/_site/feed.xml && echo "feed OK"
ls /Users/danielvaughan/Development/git/codex-blog/_site/2026/03/26/agentic-pod-roles-and-codex/index.html && echo "post OK"
```
Expected: all six lines print "OK" or the post path.

- [ ] **Step 4: Verify the new SCSS produces a real CSS file with main-site styles**

```bash
/usr/bin/grep -c 'profile-nav\|content-section\|book-card\|hero-section' /Users/danielvaughan/Development/git/codex-blog/_site/assets/css/main.css
```
Expected: ≥ 4 (one for each class — present because the main site's `_bootstrap_customization.scss` was imported).

- [ ] **Step 5: Verify the new dv-navbar markup is in rendered post HTML**

```bash
/usr/bin/grep -l 'class="profile-nav navbar navbar-expand-lg"' /Users/danielvaughan/Development/git/codex-blog/_site/index.html
/usr/bin/grep -l 'class="dv-footer' /Users/danielvaughan/Development/git/codex-blog/_site/index.html
```
Expected: both print `_site/index.html`.

- [ ] **Step 6: Smoke serve**

```bash
cd /Users/danielvaughan/Development/git/codex-blog && bundle exec jekyll serve --no-watch --detach > /tmp/jekyll-refactor.log 2>&1
sleep 7
/usr/bin/curl -sf -o /dev/null -w "home:    %{http_code}\n" http://localhost:4000/
/usr/bin/curl -sf -o /dev/null -w "tags:    %{http_code}\n" http://localhost:4000/tags/
/usr/bin/curl -sf -o /dev/null -w "feed:    %{http_code}\n" http://localhost:4000/feed.xml
/usr/bin/curl -sf -o /dev/null -w "post:    %{http_code}\n" http://localhost:4000/2026/03/26/agentic-pod-roles-and-codex/
/usr/bin/curl -sf -o /dev/null -w "broken:  %{http_code}\n" http://localhost:4000/2026/04/07/cross-model-review-loop-automation/
```
Expected: all five lines print `200`.

- [ ] **Step 7: Stop the server**

```bash
pkill -f "jekyll serve" 2>/dev/null || true
```

- [ ] **Step 8: No commit needed.** This task is verification only.

If any check fails, stop and report which check, with the failure output.

---

## Task 17: Visual browser inspection (Chrome DevTools MCP)

This task is verification only. Skip if Chrome DevTools MCP tools are unavailable.

- [ ] **Step 1: Restart the local server**

```bash
pkill -f "jekyll serve" 2>/dev/null || true
sleep 1
cd /Users/danielvaughan/Development/git/codex-blog && bundle exec jekyll serve --no-watch --detach > /tmp/jekyll-refactor.log 2>&1
sleep 7
```

- [ ] **Step 2: Open the home page in Chrome via the MCP `navigate_page` tool**

Navigate to `http://localhost:4000/`. Take a viewport screenshot. Verify visually:
- Sticky navy navbar at top with circular avatar + "Daniel Vaughan" + Home / Tags / Main site → links.
- "About this blog" intro band with grey background and a blue underline accent on the h2.
- Card list of posts using the main site's `book-card` style (left blue border, white background).
- Pagination at the bottom showing "Page 1 of N" with Older → link.
- One-line copyright footer at the bottom.
- Compare visually to `https://danielvaughan.com/` — same Montserrat headings, same primary blue, same navy navbar, same `book-card` look, same `content-section` h2 underline.

- [ ] **Step 3: Open a post page**

Navigate to `http://localhost:4000/2026/03/26/agentic-pod-roles-and-codex/`. Verify visually:
- Same navy navbar at top.
- Title in Montserrat 700.
- Date below title.
- Tag badges below the date (small blue pill chips).
- Inline TOC box with grey background and blue left border, containing a list of h2 anchors.
- Article body with blue underline accents on h2 headings, uppercase blue h3 headings, sketchnote at top of body.
- Same one-line footer at bottom.

- [ ] **Step 4: Open the previously-broken article**

Navigate to `http://localhost:4000/2026/04/07/cross-model-review-loop-automation/`. Verify:
- Same chrome (navbar, footer).
- Mermaid diagrams render as SVGs (3 of them on this article).
- Body content reads cleanly.

- [ ] **Step 5: Check console errors via `list_console_messages`**

Expected: zero `error` messages on home, post, and broken-article pages. A single 404 for a missing favicon is acceptable.

- [ ] **Step 6: Open the tags page**

Navigate to `http://localhost:4000/tags/`. Verify the navbar renders, the page lists tags alphabetically, and the page styling matches the rest of the site.

- [ ] **Step 7: Stop the server**

```bash
pkill -f "jekyll serve" 2>/dev/null || true
```

- [ ] **Step 8: No commit needed.** This task is verification only.

---

## Task 18: Update the sync script in codex-resources worktree

**Files:**
- Modify: `/Users/danielvaughan/Development/git/codex-resources-blog-sync/scripts/sync-to-codex-blog.sh`
- Modify: `/Users/danielvaughan/Development/git/codex-resources-blog-sync/scripts/test-sync-to-codex-blog.sh`

The current script rewrites `^layout: default$ → layout: single`. Change it to delete any `^layout: default$` or `^layout: single$` line entirely so the new `_config.yml` default `layout: post` applies.

- [ ] **Step 1: Read existing sync script section**

```bash
/usr/bin/grep -n "Normalising front-matter layout" /Users/danielvaughan/Development/git/codex-resources-blog-sync/scripts/sync-to-codex-blog.sh
```
Expected: returns a line number (was added in the previous round).

- [ ] **Step 2: Replace the layout normalisation block**

Edit `/Users/danielvaughan/Development/git/codex-resources-blog-sync/scripts/sync-to-codex-blog.sh`. Find:

```bash
echo "==> Normalising front-matter layout (default -> single)"
# Many articles in codex-resources were authored with `layout: default` which
# in minimal-mistakes is the bare HTML wrapper without sidebar, TOC, or post
# chrome — and breaks the theme JS because there's no <article> element.
# Rewrite to `layout: single` which is the proper post layout.
find "$POSTS_DST" -name '*.md' -type f \
  -exec sed -i.bak 's|^layout: default$|layout: single|' {} \;
find "$POSTS_DST" -name '*.bak' -type f -delete
```

Replace with:

```bash
echo "==> Stripping layout front-matter (let codex-blog _config.yml apply)"
# codex-blog now uses Jekyll defaults (layout: post for posts, layout: page
# for pages). Per-post front matter overrides defaults, so we strip any
# existing `layout: single` or `layout: default` line so the codex-blog
# default takes effect cleanly.
find "$POSTS_DST" -name '*.md' -type f \
  -exec sed -i.bak \
    -e '/^layout: default$/d' \
    -e '/^layout: single$/d' \
    {} \;
find "$POSTS_DST" -name '*.bak' -type f -delete
```

- [ ] **Step 3: Update the test harness fixture**

Edit `/Users/danielvaughan/Development/git/codex-resources-blog-sync/scripts/test-sync-to-codex-blog.sh`. The current Fixture A is:

```bash
cat > "$SRC/articles/2026-04-07-test-post.md" <<'MD'
---
title: "Test Post"
layout: default
date: 2026-04-07
tags: test
---

![Sketchnote](/codex-resources/sketchnotes/articles/2026-04-07-test-post.png)
**Date:** 2026-04-07
**Tags:** test, sample, fixtures

This is a test post body.
MD
```

Leave it as-is (it still has `layout: default` so the new strip rule applies).

- [ ] **Step 4: Update the layout assertion**

Find the assertion in `test-sync-to-codex-blog.sh`:

```bash
# Layout normalised (default -> single)
assert_no_grep "^layout: default$" "$DST/_posts/2026-04-07-test-post.md"
assert_grep    "^layout: single$" "$DST/_posts/2026-04-07-test-post.md"
```

Replace with:

```bash
# Layout line stripped entirely (codex-blog _config.yml will apply layout: post)
assert_no_grep "^layout: default$" "$DST/_posts/2026-04-07-test-post.md"
assert_no_grep "^layout: single$" "$DST/_posts/2026-04-07-test-post.md"
```

- [ ] **Step 5: Run the test harness**

```bash
bash /Users/danielvaughan/Development/git/codex-resources-blog-sync/scripts/test-sync-to-codex-blog.sh 2>&1 | tail -10
```
Expected: ends with `PASS: all assertions met`.

- [ ] **Step 6: Commit the script + harness changes**

```bash
git -C /Users/danielvaughan/Development/git/codex-resources-blog-sync add scripts/sync-to-codex-blog.sh scripts/test-sync-to-codex-blog.sh
git -C /Users/danielvaughan/Development/git/codex-resources-blog-sync commit -m "fix(sync): strip layout: single/default lines (let codex-blog defaults apply)

codex-blog has been refactored to share source code with the main site
via a submodule. Its new _config.yml sets 'layout: post' as the default
for posts. Per-post front matter overrides defaults, so the previous
normalisation that REWROTE 'layout: default' -> 'layout: single' would
lock in the wrong layout. Strip the layout line entirely instead so the
codex-blog default takes effect.

Test harness fixture and assertion updated."
```

---

## Task 19: Final cleanup and summary

- [ ] **Step 1: Verify clean working tree in both repos**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog status
git -C /Users/danielvaughan/Development/git/codex-resources-blog-sync status
```
Expected: both `nothing to commit, working tree clean`.

- [ ] **Step 2: Verify the new commit history in codex-blog**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog log --oneline | head -20
```
Expected (top of log, newest first):
```
<sha> fix(rescue): strip layout: single/default from synced posts
<sha> chore: remove obsolete masthead.html and footer.html includes
<sha> refactor(config): drop minimal-mistakes, add layout defaults, exclude _main-site
<sha> refactor(head): drop a11y JS shims, add inline TOC builder
<sha> feat(include): add dv-footer.html (Bootstrap py-5 border-top, container)
<sha> feat(include): add dv-navbar.html (uses .profile-nav from main site)
<sha> feat(layout): add page.html for static pages like /tags/
<sha> feat(layout): add home.html with paginated card list and intro band
<sha> feat(layout): add post.html with header, tags, inline TOC, body
<sha> feat(layout): add codex-blog default.html using main-site head
<sha> feat(scss): add codex-blog override layer
<sha> refactor(scss): replace MM imports with main-site Bootstrap pipeline
<sha> chore: drop jekyll-remote-theme from Gemfile
<sha> chore: add symlinks into _main-site for SCSS/includes/layouts
<sha> chore: add danielvaughan.github.io as _main-site submodule
<sha> docs: add design spec for sharing code with main site via submodule
```
followed by older history.

- [ ] **Step 3: Verify the new commit history in the worktree**

```bash
git -C /Users/danielvaughan/Development/git/codex-resources-blog-sync log --oneline | head -5
```
Expected: top commit is `fix(sync): strip layout: single/default lines (let codex-blog defaults apply)`.

- [ ] **Step 4: One final clean build**

```bash
cd /Users/danielvaughan/Development/git/codex-blog && rm -rf _site .jekyll-cache && bundle exec jekyll build 2>&1 | tail -5
```
Expected: `done in N.NNN seconds`.

- [ ] **Step 5: Report summary to the controller**

Report:
- Number of new commits in codex-blog (≥15)
- Number of new commits in the worktree (1)
- Any deviations from the plan and why
- Whether Task 17 (visual browser inspection) was performed and what it found
- Reminder to the user: push both branches and merge the worktree PR:
  ```
  git -C /Users/danielvaughan/Development/git/codex-blog push
  git -C /Users/danielvaughan/Development/git/codex-resources-blog-sync push
  ```
- Reminder to trigger the sync action manually after merging the worktree PR

---

## Out of scope (deliberately not in this plan)

- Fixing the 7 articles with Liquid template warnings (`{{ ... }}` in code blocks). Content fix in codex-resources.
- Hardening the sync script's `git add -A` to only stage `_posts/` and `sketchnotes/articles/`. Sync hygiene fix.
- Restoring search. Out of scope.
- Adding visual regression tests. Manual screenshot verification only.
- Reintroducing minimal-mistakes' related posts, share buttons, sticky right TOC sidebar, or author profile sidebar — all explicitly removed in the design.
- Migrating the home page hero from the main site's profile.html — codex-blog uses its own intro band layout.
- Updating the Mermaid version or theme.
