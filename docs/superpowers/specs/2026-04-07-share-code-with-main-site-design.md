---
title: codex-blog shares source code with the main site via a git submodule
date: 2026-04-07
status: design
---

# codex-blog ↔ main site code-sharing refactor

## Goal

Make codex-blog look and feel as close to https://danielvaughan.com as
possible by **literally sharing source files** — SCSS, includes, layouts —
between the two repos through a git submodule, instead of duplicating styles
in CSS overrides on top of minimal-mistakes.

## Motivation

The user explicitly asked for two things:

1. "I would like the blog to follow much more closely the look and feel of the
   https://danielvaughan.com/ site that has the source code at
   `/Users/danielvaughan/Development/git/danielvaughan.github.io`"
2. "please use common code when possible"

The current codex-blog uses minimal-mistakes (a different CSS framework from
the main site's Bootstrap 4 + custom Bootswatch "Minty" base) plus 500+ lines
of CSS overrides trying to mimic the main site visually. The two will never
look identical with override-based mimicry — they have different reset styles,
different spacing scales, different grid systems. The only way to genuinely
share the look is to share the SCSS source files.

## Non-goals

- Migrating any of the 7 articles with Liquid template warnings (`{{ ... }}`
  in code blocks). Content fix, separate work.
- Adding visual regression tests. Verification is manual via screenshots.
- Reintroducing minimal-mistakes' search, related posts, sticky right TOC
  sidebar, share buttons, or author profile sidebar. The user has explicitly
  accepted these losses.
- Changing the codex-resources sync workflow's trigger model.

## Approach

**Submodule the main site repo (`danielvaughan.github.io`) into codex-blog
at `_main-site/`** and reference its `_sass/`, `_includes/`, and `_layouts/`
via directory symlinks. codex-blog's `assets/css/main.scss` then imports the
main site's full SCSS pipeline directly:

```scss
@import "main-site/variables";
@import "main-site/bootstrap/bootstrap";
@import "main-site/syntax-highlighting";
@import "main-site/bootstrap-4-jekyll/bootstrap-4-jekyll";
@import "main-site/bootstrap_customization";

// codex-blog overrides go here
@import "codex-blog";
```

The first five imports are the EXACT five lines the main site uses in its own
`assets/main.scss`. codex-blog inherits everything from the main site visually
for free: same colours, same Montserrat headings, same `.book-card` style,
same `.profile-nav` style, same `.content-section h2` blue underline accent,
same buttons, same typography rhythm.

Layouts that need codex-blog-specific structure (post page with inline TOC,
home page with paginated card list) are written as small files that extend
the main site's `default.html` pattern.

minimal-mistakes is removed entirely from `Gemfile`, `_config.yml`, and the
SCSS pipeline. All `.dv-*`, `.archive__*`, `.toc__*`, `.page__*` overrides
that exist today are deleted.

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                          codex-blog/ repo                          │
│                                                                    │
│  _main-site/  ─────────────────────────────────► git submodule     │
│                                                  danielvaughan.    │
│    (read-only, never edited from codex-blog)     github.io         │
│                                                                    │
│  _includes/main-site  ──→  ../_main-site/_includes (symlink)       │
│  _layouts/main-site   ──→  ../_main-site/_layouts  (symlink)       │
│  _sass/main-site      ──→  ../_main-site/_sass     (symlink)       │
│                                                                    │
│  _includes/dv-navbar.html      ◄── codex-blog navbar               │
│  _includes/dv-footer.html      ◄── codex-blog footer               │
│  _includes/head/custom.html    ◄── Montserrat link, mermaid, TOC JS│
│                                                                    │
│  _layouts/default.html         ◄── extends main-site default       │
│  _layouts/home.html            ◄── paginated card list             │
│  _layouts/post.html            ◄── header + inline TOC + body      │
│  _layouts/page.html            ◄── for /tags/                      │
│                                                                    │
│  _sass/codex-blog.scss         ◄── slim override layer (~150 lines)│
│                                                                    │
│  assets/css/main.scss          ◄── imports main-site + codex-blog  │
│  assets/images/daniel-vaughan-avatar.jpg                           │
│                                                                    │
│  _posts/                       ◄── auto-synced (untouched)         │
│  sketchnotes/articles/         ◄── auto-synced                     │
│  index.html                                                        │
│  _pages/tags.md                                                    │
│  _config.yml                                                       │
│  Gemfile                                                           │
│  CNAME                                                             │
└────────────────────────────────────────────────────────────────────┘
```

## Repo layout after the refactor

| Path | Purpose | Source |
|---|---|---|
| `_main-site/` | Git submodule of `danielvaughan.github.io` | submodule |
| `_includes/main-site/` | Symlink → `../_main-site/_includes/` | symlink |
| `_layouts/main-site/` | Symlink → `../_main-site/_layouts/` | symlink |
| `_sass/main-site/` | Symlink → `../_main-site/_sass/` | symlink |
| `_includes/dv-navbar.html` | Sticky navy navbar | new (replaces masthead.html) |
| `_includes/dv-footer.html` | One-line copyright | new (replaces footer.html) |
| `_includes/head/custom.html` | Montserrat + mermaid + TOC builder JS | modified |
| `_layouts/default.html` | Wraps body in main-site head + dv-navbar + main + dv-footer | new |
| `_layouts/post.html` | Article: header + inline TOC + tags + body | new |
| `_layouts/home.html` | Paginated card list with intro band | new |
| `_layouts/page.html` | Static page wrapper for `/tags/` | new |
| `_sass/codex-blog.scss` | Slim override layer (post-toc, post-tags, post-card, dv-navbar, dv-footer) | new |
| `assets/css/main.scss` | Imports main-site SCSS + codex-blog overrides | rewritten |
| `_config.yml` | No `remote_theme`, layout defaults `post`/`page`, exclude `_main-site` | modified |
| `Gemfile` | No `jekyll-remote-theme` | modified |
| `_includes/masthead.html` | DELETED (replaced by dv-navbar.html) | removed |
| `_includes/footer.html` | DELETED (replaced by dv-footer.html) | removed |
| `index.html` | `layout: home` (unchanged) | unchanged |
| `_pages/tags.md` | `layout: page` (unchanged front matter, body uses site.tags) | unchanged |
| `_posts/*` | Auto-synced | unchanged (after sync script update) |
| `sketchnotes/articles/*` | Auto-synced | unchanged |
| `CNAME`, `README.md` | Unchanged | unchanged |

## Submodule mechanics

**Add the submodule:**

```bash
git submodule add https://github.com/danielvaughan/danielvaughan.github.io.git _main-site
```

The clone URL is the public HTTPS URL because GitHub Pages needs to clone the
submodule during build, and HTTPS works for public repos without auth.

**Symlinks:**

```bash
ln -s ../_main-site/_includes _includes/main-site
ln -s ../_main-site/_layouts _layouts/main-site
ln -s ../_main-site/_sass _sass/main-site
```

These are tracked in git as symlinks (`mode 120000`) and resolve to the
submodule directories at file-read time. Jekyll's safe-mode allows symlinks
to subdirectories within the source tree.

**Updating the submodule:**

```bash
git submodule update --remote _main-site
git add _main-site
git commit -m "chore: bump main-site submodule"
git push
```

**`_config.yml` exclude:**

```yaml
exclude:
  - _main-site
```

This prevents Jekyll from publishing copies of the main site at
`/codex-blog/_main-site/...`. The submodule is purely a SOURCE for SCSS,
includes, and layouts; nothing inside it ends up in `_site/`.

## SCSS pipeline

`assets/css/main.scss`:

```scss
---
# Front matter required so Jekyll processes this file as SCSS
---

@import "main-site/variables";
@import "main-site/bootstrap/bootstrap";
@import "main-site/syntax-highlighting";
@import "main-site/bootstrap-4-jekyll/bootstrap-4-jekyll";
@import "main-site/bootstrap_customization";

@import "codex-blog";
```

`_sass/codex-blog.scss` (~150 lines, the entire override layer):

```scss
// codex-blog-specific overrides on top of the main site's stack.

// ---------- Sticky navy navbar ----------
// We don't use the main site's bright-blue header.html. Our markup uses
// `class="profile-nav navbar navbar-expand-lg"` so it picks up the
// .profile-nav rules from _bootstrap_customization.scss for free. The only
// addition is the round avatar image which the main site doesn't have.
.profile-nav .navbar-avatar {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  border: 2px solid rgba(255,255,255,0.35);
  object-fit: cover;
  margin-right: 0.5rem;
}
.profile-nav .navbar-brand {
  display: inline-flex;
  align-items: center;
}

// ---------- Footer ----------
// Adds a top border colour to the main site's standard <footer> markup.
.dv-footer {
  border-top-color: #eceeef;
}

// ---------- Post page tweaks ----------
.post-header {
  margin-bottom: 2rem;
}
.post-tags {
  margin-top: 0.75rem;
}
.post-tags .badge {
  margin-right: 0.25rem;
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  background: #f0f4ff;
  color: $primary;
  border: 1px solid #d9e2ff;
  padding: 0.3em 0.7em;
  text-decoration: none;
  &:hover {
    background: $primary;
    color: #fff;
    text-decoration: none;
  }
}
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
}
.post-toc li {
  font-size: 0.95rem;
  line-height: 1.5;
}
.post-toc[hidden] {
  display: none;
}
.post-body img {
  max-width: 100%;
  height: auto;
}
.post-body > p:first-child > img:only-child {
  display: block;
  margin: 0 auto 2rem;
  border-radius: 4px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.08);
}

// ---------- Home page card list ----------
.post-card {
  // book-card is defined in main site's _bootstrap_customization.scss with
  // a left blue border, white background, padding. We just tweak the title
  // size for posts (which have longer titles than book entries).
  .book-title {
    font-size: 1.15rem;
    line-height: 1.3;
  }
  .post-excerpt {
    color: $gray-700;
    font-size: 0.95rem;
    margin-top: 0.5rem;
    margin-bottom: 0;
    line-height: 1.6;
  }
  .post-tags {
    margin-top: 0.5rem;
  }
}

// ---------- Skip-to-content link (a11y) ----------
.sr-only-focusable:focus,
.sr-only-focusable:focus-visible {
  position: fixed;
  top: 8px;
  left: 8px;
  width: auto;
  height: auto;
  padding: 10px 18px;
  margin: 0;
  overflow: visible;
  clip: auto;
  background: #0d1b4b;
  color: #fff;
  font-family: $headings-font-family;
  font-size: 14px;
  font-weight: 600;
  border-radius: 4px;
  z-index: 9999;
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

## Layouts

### `_layouts/default.html`

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

### `_layouts/post.html`

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
      <a href="{{ '/tags/#' | append: tag | slugify | relative_url }}" class="badge badge-pill">{{ tag }}</a>
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

The TOC `<nav>` starts hidden. The TOC builder JS in `_includes/head/custom.html`
populates the `<ul>` from `.post-body h2` elements at DOMContentLoaded and
removes the `hidden` attribute if any are found.

### `_layouts/home.html`

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
    <article class="book-card mb-3 post-card">
      <div class="book-title">
        <a href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
      </div>
      <div class="book-meta">
        <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%B %-d, %Y" }}</time>
      </div>
      {%- if post.tags and post.tags.size > 0 -%}
      <div class="post-tags">
        {%- for tag in post.tags limit:5 -%}
        <a href="{{ '/tags/#' | append: tag | slugify | relative_url }}" class="badge badge-pill">{{ tag }}</a>
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

### `_layouts/page.html`

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

## Includes

### `_includes/dv-navbar.html`

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

### `_includes/dv-footer.html`

```html
<footer class="dv-footer py-5 border-top">
  <div class="container">
    <p>Copyright © 2000–{{ 'now' | date: "%Y" }}. Daniel Vaughan. All rights reserved</p>
  </div>
</footer>
```

### `_includes/head/custom.html`

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Montserrat:wght@400;600;700&display=swap" rel="stylesheet">

<script>
  document.addEventListener('DOMContentLoaded', function () {
    // Build inline TOC from .post-body h2 elements
    var nav = document.querySelector('nav.post-toc');
    if (nav) {
      var ul = nav.querySelector('ul');
      var headings = document.querySelectorAll('.post-body h2');
      if (headings.length === 0) {
        nav.setAttribute('hidden', '');
      } else {
        headings.forEach(function (h) {
          var id = h.id || h.textContent.toLowerCase().replace(/[^\w]+/g, '-').replace(/^-|-$/g, '');
          h.id = id;
          var li = document.createElement('li');
          var a = document.createElement('a');
          a.href = '#' + id;
          a.textContent = h.textContent;
          li.appendChild(a);
          ul.appendChild(li);
        });
        nav.removeAttribute('hidden');
      }
    }
  });
</script>

{% if site.mermaid %}
<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@{{ site.mermaid.version | default: 10 }}/dist/mermaid.esm.min.mjs';
  document.addEventListener('DOMContentLoaded', () => {
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

The a11y JS shims (`role="main"`, TOC nav `aria-label`, pagination nav
`aria-label`) from the previous round are removed because the new layouts use
real `<main>` and explicit `aria-label` attributes natively.

## `_config.yml`

```yaml
title: "Codex Blog"
description: "Articles on agentic software engineering with Codex CLI"
url: "https://codex.danielvaughan.com"
baseurl: ""
repository: "danielvaughan/codex-blog"

permalink: /:year/:month/:day/:title/
timezone: Europe/London
markdown: kramdown

paginate: 10
paginate_path: "/page:num/"

mermaid:
  version: "10"

plugins:
  - jekyll-seo-tag
  - jekyll-sitemap
  - jekyll-feed
  - jekyll-paginate
  - jekyll-include-cache

sass:
  load_paths:
    - _sass

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
      layout: post
  - scope:
      path: "_pages"
      type: pages
    values:
      layout: page

include:
  - _pages

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

## `Gemfile`

```ruby
source "https://rubygems.org"

gem "github-pages", group: :jekyll_plugins
gem "webrick"

group :jekyll_plugins do
  gem "jekyll-seo-tag"
  gem "jekyll-sitemap"
  gem "jekyll-feed"
  gem "jekyll-paginate"
  gem "jekyll-include-cache"
end
```

`jekyll-remote-theme` is removed.

## Sync script update

The sync script in codex-resources currently rewrites `^layout: default$ →
layout: single`. After this refactor we want NO `layout:` line in synced
posts so the new `_config.yml` default `layout: post` applies cleanly.

Change the rewrite to **delete any `^layout:` line** that has the value
`default` or `single`:

```bash
echo "==> Stripping layout front matter (default/single -> let _config.yml apply)"
find "$POSTS_DST" -name '*.md' -type f \
  -exec sed -i.bak \
    -e '/^layout: default$/d' \
    -e '/^layout: single$/d' \
    {} \;
find "$POSTS_DST" -name '*.bak' -type f -delete
```

Test harness fixture updated to assert that:
- `^layout: default$` is removed
- `^layout: single$` is removed
- The post gets `layout: post` applied at build time (asserted indirectly via
  Jekyll build of a fixture site, OR just by checking front matter has no
  layout key after sync)

## Migration sequence (becomes the implementation plan)

1. Add the submodule and verify file paths resolve.
2. Create the symlinks.
3. Replace `assets/css/main.scss` with the new imports.
4. Create `_sass/codex-blog.scss` with the slim override layer.
5. Create the four new layouts (`default.html`, `post.html`, `home.html`, `page.html`).
6. Create `_includes/dv-navbar.html` and `_includes/dv-footer.html`.
7. Update `_includes/head/custom.html` (add TOC builder, drop a11y shims).
8. Update `_config.yml` (drop minimal-mistakes, add layout defaults, exclude `_main-site`).
9. Update `Gemfile` (drop `jekyll-remote-theme`).
10. Delete `_includes/masthead.html` and the old `_includes/footer.html`.
11. Update sync script in codex-resources worktree (strip layout line entirely, update test harness).
12. Apply local rescue commit (strip layout lines from existing `_posts/*.md`).
13. Local clean build, smoke test 5+ representative articles + home + tags page, verify no console errors.
14. Commit codex-blog changes in two clean commits (chrome refactor + rescue).
15. Push the worktree branch and merge into codex-resources main.
16. Push codex-blog. Trigger a fresh sync action run. Verify live.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| GitHub Pages doesn't process the submodule | Both repos are public, submodule URL is HTTPS clone URL. GitHub Pages supports public submodules natively. |
| Symlinks blocked by Jekyll safe-mode | Symlinks point to subdirectories WITHIN the source tree (the submodule is inside the codex-blog repo). Safe-mode allows this. |
| SCSS variable collisions | We use the main site's variables directly (`$primary`, `$body-color`, `$gray-100..900`, `$headings-font-family`). No `$dv-*` redefinitions. |
| Front matter mismatch (synced posts have `layout: single`, defaults can't override per-post values) | Sync script update strips `layout:` lines entirely; one-shot rescue commit applies the same strip locally to existing `_posts/*.md`. |
| Bootstrap-vs-kramdown styling clashes (e.g. `h2` margin, `pre` font-size, `table` border) | Manually verify on a long article with code blocks, tables, blockquotes, footnotes, mermaid before pushing. |
| Pagination breaks (jekyll-paginate v1 only paginates `index.html`) | Home page is at `index.html` with `layout: home` — already correct. |
| Loss of TOC sidebar reduces navigation utility | Inline TOC at top of article via JS replaces it. Hidden if no h2s. |
| Loss of search modal | User explicitly accepted. Search re-add is out of scope. |
| Submodule update process is fiddly | Documented in README. Bumping is one command. |

## Acceptance criteria

The refactor is done when:

1. `bundle exec jekyll build` completes with no SCSS errors and no Liquid
   errors beyond the pre-existing warnings on the 7 articles with `{{ ... }}`
   in code blocks.
2. Browsing the home page at `https://codex.danielvaughan.com/` shows the
   sticky navy navbar, the "About this blog" intro band with grey background
   and blue underline, a paginated card list of posts using the same
   `book-card` style as the main site, and a one-line copyright footer that
   matches the main site's footer exactly (no teal band, no card-in-a-sea).
3. Browsing any individual post shows the title, date, tag badges, an inline
   TOC of h2 headings (or no TOC if the post has none), and the article body
   in Bootstrap's `.container.page-content` width.
4. Mermaid diagrams render as SVGs.
5. Tag index page at `/tags/` lists all tags alphabetically with posts under
   each.
6. Zero console errors on home, post, and tags pages.
7. Visual side-by-side comparison against `https://danielvaughan.com/` shows
   the SAME font (Montserrat 700 for headings), the SAME colours (`#0869ff`
   primary, `#323232` body, `#0d1b4b` navy, `#f8f9fa` bg-alt), the SAME
   navbar style, the SAME footer style, the SAME `.book-card` and
   `.content-section` patterns.
