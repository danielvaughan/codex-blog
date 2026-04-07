# codex-blog Reskin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restyle codex-blog (https://codex.danielvaughan.com) to look like a natural extension of danielvaughan.com — same navy sticky navbar, Montserrat headings, blue underline section accents, one-line footer — while keeping all minimal-mistakes layouts intact.

**Architecture:** Pure CSS overrides + 2 small Jekyll include overrides (`_includes/masthead.html`, `_includes/footer.html`) + 1 image asset + small edits to `index.html` and `_includes/head/custom.html`. No theme replacement, no Bootstrap import, no changes to post structure or to the sync flow.

**Tech Stack:** Jekyll 3.x via the `github-pages` gem, `minimal-mistakes@4.26.2` remote theme, SCSS, Montserrat from Google Fonts.

**Spec:** [`docs/superpowers/specs/2026-04-07-codex-blog-reskin-design.md`](../specs/2026-04-07-codex-blog-reskin-design.md)

**Reference source:** `/Users/danielvaughan/Development/git/danielvaughan.github.io/` — design tokens are ported from `_sass/_variables.scss` and `_sass/_bootstrap_customization.scss`.

---

## Pre-flight: working state

This work happens entirely in `/Users/danielvaughan/Development/git/codex-blog`. No worktree needed — codex-blog is already a clean git repo on `main`. The codex-resources sync script and workflow are NOT touched. Posts are NOT re-synced.

Each task ends with a clean working tree and a passing `bundle exec jekyll build`. After all tasks, the user pushes a single batch of commits to GitHub Pages.

**Before starting:** verify the working tree is clean.

```bash
git -C /Users/danielvaughan/Development/git/codex-blog status
```
Expected: `nothing to commit, working tree clean`. If not, stop and ask the controller what to do — the sync script's `git add -A` will swallow any uncommitted changes if a sync runs concurrently.

---

## File Structure

### Files created in this work

| Path | Purpose |
|---|---|
| `_includes/masthead.html` | Custom sticky navy navbar — overrides minimal-mistakes' default masthead |
| `_includes/footer.html` | One-line copyright footer — overrides minimal-mistakes' default footer |
| `assets/images/daniel-vaughan-avatar.jpg` | 32px+ circular brand avatar in the navbar (sourced from main site repo) |

### Files modified in this work

| Path | What changes |
|---|---|
| `assets/css/main.scss` | Add ~200 lines of "danielvaughan.com tokens" SCSS at the bottom |
| `_includes/head/custom.html` | Add Montserrat Google Fonts link above the existing mermaid block |
| `index.html` | Add `dv-intro` section above the post list; set `author_profile: false` |

### Files deleted in this work

| Path | Why |
|---|---|
| `_data/navigation.yml` | Unused after the masthead override (the new navbar hardcodes Home / Tags / Main site →) |

### Files NOT touched

- `_pages/tags.md`, `_posts/*.md`, `sketchnotes/articles/*` — content stays as-is
- `_config.yml`, `Gemfile`, `CNAME`, `.gitignore`, `README.md` — unchanged
- Anything in `/Users/danielvaughan/Development/git/codex-resources/` and `/Users/danielvaughan/Development/git/codex-resources-blog-sync/` — sync flow unchanged
- The `SYNCED-DO-NOT-EDIT.md` markers — preserved

---

## Task 1: Copy avatar from main site

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/assets/images/daniel-vaughan-avatar.jpg`

- [ ] **Step 1: Verify source image exists**

```bash
ls -lh /Users/danielvaughan/Development/git/danielvaughan.github.io/img/daniel-vaughan-en.jpg
```
Expected: file exists, size shown. If missing, stop and report BLOCKED — the avatar source is gone.

- [ ] **Step 2: Copy the image**

```bash
mkdir -p /Users/danielvaughan/Development/git/codex-blog/assets/images
cp /Users/danielvaughan/Development/git/danielvaughan.github.io/img/daniel-vaughan-en.jpg \
   /Users/danielvaughan/Development/git/codex-blog/assets/images/daniel-vaughan-avatar.jpg
```

- [ ] **Step 3: Verify**

```bash
file /Users/danielvaughan/Development/git/codex-blog/assets/images/daniel-vaughan-avatar.jpg
```
Expected: `JPEG image data, ...` with non-zero dimensions.

- [ ] **Step 4: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add assets/images/daniel-vaughan-avatar.jpg
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "chore: add brand avatar from danielvaughan.com main site"
```

---

## Task 2: Add Montserrat font loading

**Files:**
- Modify: `/Users/danielvaughan/Development/git/codex-blog/_includes/head/custom.html`

The file already exists with the mermaid loader from prior work. Read it first to confirm exact existing content, then prepend the Montserrat link block.

- [ ] **Step 1: Read the existing file**

```bash
cat /Users/danielvaughan/Development/git/codex-blog/_includes/head/custom.html
```
Expected: starts with `{% if site.mermaid %}` and contains the mermaid script tag. If different, stop and report — the prepend approach assumes the existing structure.

- [ ] **Step 2: Replace the file with the new content**

Overwrite `/Users/danielvaughan/Development/git/codex-blog/_includes/head/custom.html` with:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Montserrat:wght@400;600;700&display=swap" rel="stylesheet">

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

- [ ] **Step 3: Verify build succeeds**

```bash
cd /Users/danielvaughan/Development/git/codex-blog && bundle exec jekyll build 2>&1 | tail -5
```
Expected: build completes, no new errors. The Montserrat link will appear in `_site/<any-page>/index.html` `<head>`.

- [ ] **Step 4: Verify Montserrat link rendered in output**

```bash
grep -l "fonts.googleapis.com/css2?family=Montserrat" /Users/danielvaughan/Development/git/codex-blog/_site/index.html
```
Expected: prints `_site/index.html` (file path is grep's match output).

- [ ] **Step 5: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add _includes/head/custom.html
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "feat: load Montserrat font from Google Fonts"
```

---

## Task 3: Override masthead with custom navbar

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/_includes/masthead.html`

Jekyll resolves includes from the local repo before the remote theme. Creating `_includes/masthead.html` in codex-blog completely replaces minimal-mistakes' default masthead. minimal-mistakes' `default.html` layout includes `masthead.html` by name, so this override applies to every page.

- [ ] **Step 1: Create the file**

Write `/Users/danielvaughan/Development/git/codex-blog/_includes/masthead.html` with EXACTLY this content:

```html
<nav class="dv-nav" role="navigation" aria-label="Main">
  <div class="dv-nav__inner">
    <a class="dv-nav__brand" href="{{ '/' | relative_url }}">
      <img class="dv-nav__avatar"
           src="{{ '/assets/images/daniel-vaughan-avatar.jpg' | relative_url }}"
           alt="" width="32" height="32">
      <span>Daniel Vaughan</span>
    </a>
    <ul class="dv-nav__links">
      <li><a href="{{ '/' | relative_url }}">Home</a></li>
      <li><a href="{{ '/tags/' | relative_url }}">Tags</a></li>
      <li><a href="https://danielvaughan.com/" rel="noopener">Main site →</a></li>
    </ul>
  </div>
</nav>
```

- [ ] **Step 2: Verify build still succeeds**

```bash
cd /Users/danielvaughan/Development/git/codex-blog && bundle exec jekyll build 2>&1 | tail -5
```
Expected: build completes, no new errors.

- [ ] **Step 3: Verify the new navbar markup is in the rendered output**

```bash
grep -l 'class="dv-nav"' /Users/danielvaughan/Development/git/codex-blog/_site/index.html
grep -c 'Main site' /Users/danielvaughan/Development/git/codex-blog/_site/index.html
```
Expected: first command prints the file path; second command prints `1` or more.

- [ ] **Step 4: Verify the old masthead markup is gone**

```bash
grep -c 'class="masthead"' /Users/danielvaughan/Development/git/codex-blog/_site/index.html || echo "0"
```
Expected: `0`. (The minimal-mistakes default masthead used the class `masthead` — we have replaced it entirely.)

- [ ] **Step 5: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add _includes/masthead.html
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "feat: replace minimal-mistakes masthead with custom navy navbar"
```

---

## Task 4: Override footer with one-line copyright

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/_includes/footer.html`

Same Jekyll local-first include resolution as Task 3.

- [ ] **Step 1: Create the file**

Write `/Users/danielvaughan/Development/git/codex-blog/_includes/footer.html` with EXACTLY this content:

```html
<footer class="dv-footer">
  <div class="dv-footer__inner">
    Copyright © 2000–{{ 'now' | date: "%Y" }}. Daniel Vaughan. All rights reserved.
  </div>
</footer>
```

- [ ] **Step 2: Verify build still succeeds**

```bash
cd /Users/danielvaughan/Development/git/codex-blog && bundle exec jekyll build 2>&1 | tail -5
```
Expected: build completes, no new errors.

- [ ] **Step 3: Verify footer appears in output**

```bash
grep -l 'class="dv-footer"' /Users/danielvaughan/Development/git/codex-blog/_site/index.html
grep 'Daniel Vaughan. All rights reserved' /Users/danielvaughan/Development/git/codex-blog/_site/index.html | head -1
```
Expected: first command prints the file path; second command prints the copyright line.

- [ ] **Step 4: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add _includes/footer.html
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "feat: replace minimal-mistakes footer with one-line copyright"
```

---

## Task 5: Add intro band to index.html

**Files:**
- Modify: `/Users/danielvaughan/Development/git/codex-blog/index.html`

The current file is just front matter (no body) — `layout: home` from minimal-mistakes renders the post list automatically. We add an intro band as page content; minimal-mistakes' `home` layout renders page content above the post list.

- [ ] **Step 1: Read existing index.html**

```bash
cat /Users/danielvaughan/Development/git/codex-blog/index.html
```
Expected: current content is:
```
---
title: "Codex Blog"
layout: home
author_profile: true
---
```
(Empty body.)

- [ ] **Step 2: Replace with new content**

Overwrite `/Users/danielvaughan/Development/git/codex-blog/index.html` with EXACTLY this:

```html
---
title: "Codex Blog"
layout: home
author_profile: false
---

<section class="dv-intro">
  <div class="dv-intro__inner">
    <h2>About this blog</h2>
    <p>Notes on agentic software engineering, Codex CLI, and how AI is changing the way we build software. Published as I write them.</p>
  </div>
</section>
```

Two changes vs the original: `author_profile: false` (was `true`), and the new `<section class="dv-intro">` body.

- [ ] **Step 3: Verify build still succeeds**

```bash
cd /Users/danielvaughan/Development/git/codex-blog && bundle exec jekyll build 2>&1 | tail -5
```
Expected: build completes, no new errors.

- [ ] **Step 4: Verify intro section is in output**

```bash
grep 'class="dv-intro"' /Users/danielvaughan/Development/git/codex-blog/_site/index.html
grep 'About this blog' /Users/danielvaughan/Development/git/codex-blog/_site/index.html | head -1
```
Expected: both grep commands print matching lines.

- [ ] **Step 5: Verify post list still renders below the intro**

```bash
grep -c 'archive__item' /Users/danielvaughan/Development/git/codex-blog/_site/index.html
```
Expected: a number ≥ 10 (one per post entry on the first page).

- [ ] **Step 6: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add index.html
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "feat: add intro band to home page above post list"
```

---

## Task 6: Delete unused navigation.yml

**Files:**
- Delete: `/Users/danielvaughan/Development/git/codex-blog/_data/navigation.yml`

The custom masthead from Task 3 hardcodes its three links. `_data/navigation.yml` was used by the default minimal-mistakes masthead which is now overridden. The file is unreferenced; deleting it prevents future confusion.

- [ ] **Step 1: Verify file exists**

```bash
ls /Users/danielvaughan/Development/git/codex-blog/_data/navigation.yml
```
Expected: file exists.

- [ ] **Step 2: Delete the file with git rm (stages the deletion automatically)**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog rm _data/navigation.yml
```
Expected: prints `rm '_data/navigation.yml'`.

- [ ] **Step 3: Remove the now-empty directory if it still exists**

```bash
rmdir /Users/danielvaughan/Development/git/codex-blog/_data 2>/dev/null || true
```

(Empty directories are not tracked by git, so this is purely housekeeping; if the directory still has files for some reason, the `rmdir` fails silently and leaves it alone.)

- [ ] **Step 4: Verify build still succeeds**

```bash
cd /Users/danielvaughan/Development/git/codex-blog && bundle exec jekyll build 2>&1 | tail -5
```
Expected: build completes, no new errors.

- [ ] **Step 5: Commit the deletion**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "chore: remove unused _data/navigation.yml after masthead override"
```

---

## Task 7: Add danielvaughan.com SCSS tokens

**Files:**
- Modify: `/Users/danielvaughan/Development/git/codex-blog/assets/css/main.scss`

The file already exists with line-height tweaks. Append a new "danielvaughan.com tokens" block at the bottom. This is the longest task by file size; it contains all the styling for the new navbar, footer, intro band, post page restyle, and post list restyle.

- [ ] **Step 1: Read the existing file**

```bash
cat /Users/danielvaughan/Development/git/codex-blog/assets/css/main.scss
```
Expected: starts with `---` front matter and `@charset "utf-8";`, imports the minimal-mistakes skin and theme, then has the existing line-height tweak at the bottom.

- [ ] **Step 2: Replace the file with the extended version**

Overwrite `/Users/danielvaughan/Development/git/codex-blog/assets/css/main.scss` with EXACTLY this content (the existing imports and line-height block are preserved verbatim; the new tokens block is appended):

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

// =============================================================================
// danielvaughan.com tokens — ported from main site _bootstrap_customization.scss
// =============================================================================

$dv-primary:  #0869ff;
$dv-navy:     #0d1b4b;
$dv-body:     #323232;
$dv-heading:  #343a40;
$dv-bg-alt:   #f8f9fa;
$dv-border:   #e9ecef;
$dv-muted:    #888;
$dv-font-headings: "Montserrat", -apple-system, system-ui, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;

// ----- Sticky navy navbar (replaces minimal-mistakes masthead) -----

.dv-nav {
  background: $dv-navy;
  position: sticky;
  top: 0;
  z-index: 1000;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
}
.dv-nav__inner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  max-width: 1024px;
  margin: 0 auto;
  padding: 0.4rem 1.25rem;
}
.dv-nav__brand {
  display: inline-flex;
  align-items: center;
  gap: 0.6rem;
  color: #fff;
  font-family: $dv-font-headings;
  font-weight: 700;
  font-size: 1rem;
  text-decoration: none;
  &:hover { color: rgba(255,255,255,0.85); text-decoration: none; }
}
.dv-nav__avatar {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  border: 2px solid rgba(255,255,255,0.35);
  object-fit: cover;
}
.dv-nav__links {
  display: flex;
  gap: 0.25rem;
  list-style: none;
  margin: 0;
  padding: 0;
  li a {
    display: inline-block;
    padding: 0.4rem 0.85rem;
    color: rgba(255,255,255,0.75);
    font-family: $dv-font-headings;
    font-size: 0.8rem;
    font-weight: 600;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    text-decoration: none;
    &:hover { color: #fff; text-decoration: none; }
  }
}
@media (max-width: 600px) {
  .dv-nav__inner { flex-direction: column; gap: 0.5rem; padding: 0.6rem 1rem; }
  .dv-nav__links { flex-wrap: wrap; justify-content: center; }
}

// ----- Home page intro band -----

.dv-intro {
  background: $dv-bg-alt;
  border-bottom: 1px solid $dv-border;
  padding: 2.5rem 0 2rem;
  margin-bottom: 2rem;
}
.dv-intro__inner {
  max-width: 1024px;
  margin: 0 auto;
  padding: 0 1.25rem;
}
.dv-intro h2 {
  font-family: $dv-font-headings;
  font-size: 1.55rem;
  font-weight: 700;
  color: $dv-heading;
  margin: 0 0 0.75rem;
  padding-bottom: 0.6rem;
  border-bottom: 3px solid $dv-primary;
  display: inline-block;
}
.dv-intro p {
  font-size: 1rem;
  line-height: 1.75;
  color: $dv-body;
  margin: 0;
  max-width: 60ch;
}

// ----- Footer -----

.dv-footer {
  border-top: 1px solid $dv-border;
  background: #ffffff;
  padding: 2.5rem 0;
  margin-top: 3rem;
  text-align: center;
}
.dv-footer__inner {
  max-width: 1024px;
  margin: 0 auto;
  padding: 0 1.25rem;
  font-family: $dv-font-headings;
  font-size: 0.82rem;
  color: $dv-muted;
}

// ----- Post page restyle (CSS overrides on minimal-mistakes selectors) -----

.page__title {
  font-family: $dv-font-headings;
  font-weight: 700;
  color: $dv-heading;
  letter-spacing: -0.01em;
}

.page__content {
  h2, h3, h4, h5, h6 {
    font-family: $dv-font-headings;
    color: $dv-heading;
  }
  h2 {
    font-size: 1.55rem;
    font-weight: 700;
    padding-bottom: 0.6rem;
    border-bottom: 3px solid $dv-primary;
    display: inline-block;
    margin-top: 2.5rem;
  }
  h3 {
    font-size: 1.05rem;
    font-weight: 700;
    color: $dv-primary;
    text-transform: uppercase;
    letter-spacing: 0.04em;
    margin-top: 2rem;
  }
  a { color: $dv-primary; }
  a:hover { color: $dv-navy; }
}

// TOC sidebar
.toc {
  .nav__title {
    background: $dv-navy;
    color: #fff;
    font-family: $dv-font-headings;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.04em;
  }
  .toc__menu a {
    font-family: $dv-font-headings;
    font-size: 0.85em;
  }
  .toc__menu .active a {
    background: #f0f4ff;
    color: $dv-primary;
    border-left: 3px solid $dv-primary;
  }
}

// Author profile sidebar
.author__name { font-family: $dv-font-headings; font-weight: 700; }
.author__bio  { font-size: 0.9rem; }

// Post meta (read time, tags, share)
.page__meta,
.page__taxonomy {
  font-family: $dv-font-headings;
  font-size: 0.78rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: $dv-muted;
}

// ----- Home page post list restyle -----

.archive__item-title {
  font-family: $dv-font-headings;
  font-weight: 700;
  a { color: $dv-heading; }
  a:hover { color: $dv-primary; text-decoration: none; }
}
.archive__item-excerpt {
  font-size: 0.95rem;
  color: #5a5a5a;
}
```

- [ ] **Step 3: Verify SCSS compiles**

```bash
cd /Users/danielvaughan/Development/git/codex-blog && rm -rf _site .jekyll-cache && bundle exec jekyll build 2>&1 | tail -10
```
Expected: build completes. The line `done in N.NNN seconds` appears. No SCSS compile errors. Pre-existing Liquid template warnings on the 7 affected articles are still present and expected.

- [ ] **Step 4: Verify the new tokens are in the compiled CSS**

```bash
grep -c "dv-nav\|dv-intro\|dv-footer" /Users/danielvaughan/Development/git/codex-blog/_site/assets/css/main.css
```
Expected: a number ≥ 10 (multiple selectors across the three components).

- [ ] **Step 5: Commit**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog add assets/css/main.scss
git -C /Users/danielvaughan/Development/git/codex-blog commit -m "feat: add danielvaughan.com design tokens to main.scss"
```

---

## Task 8: Visual smoke test (local serve)

This task is verification only — it produces no commits.

- [ ] **Step 1: Stop any existing local server**

```bash
pkill -f "jekyll serve" 2>/dev/null || true
sleep 1
```

- [ ] **Step 2: Start a fresh local server**

```bash
cd /Users/danielvaughan/Development/git/codex-blog && \
  bundle exec jekyll serve --no-watch --detach > /tmp/jekyll-reskin.log 2>&1
sleep 6
```

- [ ] **Step 3: Smoke test HTTP responses**

```bash
curl -sf -o /dev/null -w "home:    HTTP %{http_code}\n" http://localhost:4000/
curl -sf -o /dev/null -w "tags:    HTTP %{http_code}\n" http://localhost:4000/tags/
curl -sf -o /dev/null -w "feed:    HTTP %{http_code}\n" http://localhost:4000/feed.xml
curl -sf -o /dev/null -w "post:    HTTP %{http_code}\n" http://localhost:4000/2026/03/26/agentic-pod-roles-and-codex/
curl -sf -o /dev/null -w "broken:  HTTP %{http_code}\n" http://localhost:4000/2026/04/07/cross-model-review-loop-automation/
```
Expected: all five lines print `HTTP 200`.

- [ ] **Step 4: Verify the navbar renders on all pages**

```bash
for path in / /tags/ /2026/03/26/agentic-pod-roles-and-codex/ /2026/04/07/cross-model-review-loop-automation/; do
  printf "%-60s " "$path"
  curl -sf "http://localhost:4000${path}" | grep -q 'class="dv-nav"' && echo "navbar OK" || echo "navbar MISSING"
done
```
Expected: every line ends with `navbar OK`.

- [ ] **Step 5: Verify the intro band only on home**

```bash
curl -sf http://localhost:4000/ | grep -q 'class="dv-intro"' && echo "home: intro OK" || echo "home: intro MISSING"
curl -sf http://localhost:4000/tags/ | grep -q 'class="dv-intro"' && echo "tags: intro PRESENT (should be missing)" || echo "tags: no intro OK"
```
Expected: `home: intro OK` and `tags: no intro OK`.

- [ ] **Step 6: Verify the footer is the new one-liner**

```bash
curl -sf http://localhost:4000/ | grep -q 'class="dv-footer"' && echo "footer OK" || echo "footer MISSING"
curl -sf http://localhost:4000/ | grep -q 'All rights reserved' && echo "copyright OK" || echo "copyright MISSING"
```
Expected: both `OK`.

- [ ] **Step 7: Verify Montserrat is loaded**

```bash
curl -sf http://localhost:4000/ | grep -q 'fonts.googleapis.com/css2?family=Montserrat' && echo "Montserrat link OK" || echo "Montserrat link MISSING"
```
Expected: `Montserrat link OK`.

- [ ] **Step 8: Verify mermaid still works on a known mermaid post**

```bash
curl -sf "http://localhost:4000/2026/04/07/cross-model-review-loop-automation/" | grep -c 'language-mermaid' || echo "0"
```
Expected: a number ≥ 1 (the raw kramdown output before the JS converts it client-side; we are checking the page renders without errors and contains the unconverted mermaid blocks that the client script will pick up).

- [ ] **Step 9: Stop the local server**

```bash
pkill -f "jekyll serve" 2>/dev/null || true
```

- [ ] **Step 10: No commit needed.** This task is verification only.

If any step fails, stop and report which step. Do not move on to the visual browser inspection in Task 9 until smoke tests pass.

---

## Task 9: Visual browser inspection

This task is verification using Chrome DevTools MCP. It produces no commits. Skip this task if Chrome DevTools MCP tools are unavailable.

The goal is to confirm the rendered pages visually match the design intent — navbar styling, fonts, colours, intro band, footer.

- [ ] **Step 1: Start the local server**

```bash
pkill -f "jekyll serve" 2>/dev/null || true
sleep 1
cd /Users/danielvaughan/Development/git/codex-blog && \
  bundle exec jekyll serve --no-watch --detach > /tmp/jekyll-reskin.log 2>&1
sleep 6
```

- [ ] **Step 2: Open the home page in Chrome via the MCP `new_page` or `navigate_page` tool**

Navigate to `http://localhost:4000/`. Take a viewport screenshot. Verify visually:
- Sticky navy navbar at top with circular avatar + "Daniel Vaughan" brand + 3 nav links.
- Intro band below the navbar with grey background, "About this blog" h2 with blue underline, one-paragraph description.
- Post list below the intro band with Montserrat post titles.
- One-line copyright footer at bottom.

If any visual element is missing or wrong, stop and report.

- [ ] **Step 3: Open a post page**

Navigate to `http://localhost:4000/2026/04/07/cross-model-review-loop-automation/`. Take a screenshot. Verify visually:
- Same navbar at top.
- Post title in Montserrat.
- TOC sidebar on the right with a navy header bar and Montserrat links.
- Body h2s show the blue underline accent.
- Mermaid diagrams render as SVG flowcharts (not raw code blocks).
- Same one-line footer at bottom.

- [ ] **Step 4: Check console errors via the MCP `list_console_messages` tool**

Expected: zero `error` messages. A single 404 for a missing favicon is acceptable. Any other JS errors must be investigated before declaring the task done.

- [ ] **Step 5: Check the tags page**

Navigate to `http://localhost:4000/tags/`. Verify the navbar renders, the page lists tags alphabetically, no intro band (because `dv-intro` lives only in `index.html`), one-line footer.

- [ ] **Step 6: Stop the server**

```bash
pkill -f "jekyll serve" 2>/dev/null || true
```

- [ ] **Step 7: No commit needed.** This task is verification only.

---

## Task 10: Final cleanup and summary

- [ ] **Step 1: Verify clean working tree**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog status
```
Expected: `nothing to commit, working tree clean`.

- [ ] **Step 2: Verify the new commit history**

```bash
git -C /Users/danielvaughan/Development/git/codex-blog log --oneline | head -10
```
Expected (top 7 commits, newest first): roughly something like
```
<sha> feat: add danielvaughan.com design tokens to main.scss
<sha> chore: remove unused _data/navigation.yml after masthead override
<sha> feat: add intro band to home page above post list
<sha> feat: replace minimal-mistakes footer with one-line copyright
<sha> feat: replace minimal-mistakes masthead with custom navy navbar
<sha> feat: load Montserrat font from Google Fonts
<sha> chore: add brand avatar from danielvaughan.com main site
```
followed by the prior reskin spec doc commit and the prior history.

- [ ] **Step 3: One final clean build**

```bash
cd /Users/danielvaughan/Development/git/codex-blog && \
  rm -rf _site .jekyll-cache && \
  bundle exec jekyll build 2>&1 | tail -5
```
Expected: `done in N.NNN seconds`.

- [ ] **Step 4: Report summary to controller**

Report:
- Number of new commits added by this work.
- Any deviations from the plan and why.
- Visual inspection results from Task 9 (or a note if Task 9 was skipped).
- Reminder to the user: push the codex-blog branch to deploy:
  ```
  git -C /Users/danielvaughan/Development/git/codex-blog push
  ```

---

## Out of scope (deliberately not in this plan)

- Fixing the 7 articles with Liquid template warnings on `{{ ... }}` code blocks (`workflow-md`, `dbt-data-engineering`, `github-actions-complete-integration-guide`, `java-spring-boot-teams`, `rust-teams`, `triggers-event-driven-github-automation`, and one more). Content fix in codex-resources, separate work.
- Hardening the sync script's `git add -A` to only stage `_posts/` and `sketchnotes/articles/`. Sync hygiene fix in codex-resources, separate work.
- Restoring search. The minimal-mistakes search modal trigger lived in the default masthead which we just replaced. Adding it back means wiring a `<button class="search__toggle">🔍</button>` into `.dv-nav__links` and verifying the underlying modal still mounts. Out of scope.
- Restyling code blocks, blockquotes, or tables. The minimal-mistakes defaults are legible and the main site has no equivalent visual treatments to copy.
- Restyling pagination buttons (they still use minimal-mistakes' default styling). Could be a small follow-up if visually jarring.
