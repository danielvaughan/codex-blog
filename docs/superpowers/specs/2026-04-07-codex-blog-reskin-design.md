---
title: codex-blog visual reskin to match danielvaughan.com
date: 2026-04-07
status: design
---

# codex-blog Reskin Design

## Goal

Make `codex-blog` (https://codex.danielvaughan.com) feel like a natural
extension of the personal site at https://danielvaughan.com — same colour
palette, same Montserrat heading font, same sticky navy navbar, same blue
underline section accent, same one-line footer — without abandoning the
minimal-mistakes layouts that already give us a working post page (TOC sidebar,
mermaid rendering, syntax highlighting, sketchnote at top).

## Non-goals

- Replacing minimal-mistakes with a custom Bootstrap-based theme. The two
  CSS frameworks would conflict and we would lose the post layout we already
  fixed.
- Adding a marketing-style hero to the home page. The user wants
  "branded but blog-first" — content density wins over splash.
- Fixing content issues in source articles (the 7 articles with Liquid
  template warnings on `{{ ... }}` code blocks). Separate content concern.
- Hardening the sync script (`git add -A` swallowing hand-authored files).
  Separate sync hygiene concern.
- Re-syncing posts. The reskin is purely CSS + a few hand-authored HTML
  overrides; existing post content is correct and renders through the new
  styles unchanged.

## Reference

The main site source lives at
`/Users/danielvaughan/Development/git/danielvaughan.github.io/`.
Design tokens are ported from:

- `_sass/_variables.scss` — colours, body, headings font family
- `_sass/_bootstrap_customization.scss:332-373` — `.profile-nav` (sticky navy
  navbar)
- `_sass/_bootstrap_customization.scss:421-462` — `.content-section` (h2 with
  blue underline accent, h3 with uppercase blue treatment)
- `_layouts/profile.html` — navbar markup pattern
- `_includes/footer.html` — one-line copyright footer

The main site uses Bootstrap 4 + a custom Bootswatch "Minty" base. We are
NOT importing any of that. We are reproducing the visual outcome by lifting
the hex values and rule blocks into codex-blog's existing minimal-mistakes
SCSS overrides.

## Approach

Keep all minimal-mistakes layouts intact. Override the chrome via:

1. **CSS overrides** in the existing `assets/css/main.scss` — the bulk of the
   reskin lives here as a clearly-marked "danielvaughan.com tokens" block.
2. **One Google Fonts link** in the existing `_includes/head/custom.html` —
   loads Montserrat alongside the existing mermaid script tag.
3. **Two new include files**, `_includes/masthead.html` and
   `_includes/footer.html`, which Jekyll resolves from the local repo before
   the remote theme. These completely replace minimal-mistakes' default
   masthead and footer.
4. **One image asset** copied from the main site repo —
   `assets/images/daniel-vaughan-avatar.jpg`.
5. **Two small content edits**: `index.html` gets an intro band above the
   post list; `_data/navigation.yml` gets deleted because the new masthead
   hardcodes its three links.

The post layout itself — `single` from minimal-mistakes — is NOT touched
structurally. TOC sidebar, author profile sidebar, sketchnote header,
syntax-highlighted code blocks, mermaid rendering, share buttons, related
posts: all preserved.

## Design tokens

Lifted verbatim from `danielvaughan.github.io/_sass/_variables.scss` and
`_bootstrap_customization.scss`.

**Colours:**

| Token | Value | Used for |
|---|---|---|
| `primary blue` | `#0869ff` | Links, accents, h2 underline, h3 text, TOC active item |
| `navy` | `#0d1b4b` | Sticky navbar background, post-page TOC heading bar, link hover |
| `body text` | `#323232` | Body paragraphs |
| `heading text` | `#343a40` | h1, h2, h4-h6 |
| `bg-alt` | `#f8f9fa` | Home page intro band background |
| `border` | `#e9ecef` | Footer top border, intro band bottom border |
| `muted` | `#888` | Footer text, post meta, dates |

**Typography:**

| Element | Family | Size | Weight | Notes |
|---|---|---|---|---|
| Body | system stack (unchanged) | 16px | 400 | minimal-mistakes default |
| All headings | Montserrat | varies | 700 | Loaded from Google Fonts |
| Navbar brand | Montserrat | 1rem | 700 | white |
| Navbar links | Montserrat | 0.8rem | 600 | white-translucent, uppercase, letter-spacing 0.06em |
| Section h2 (intro band, post body) | Montserrat | 1.55rem | 700 | with `border-bottom: 3px solid #0869ff; display: inline-block;` |
| Section h3 (post body) | Montserrat | 1.05rem | 700 | colour `#0869ff`, uppercase, letter-spacing 0.04em |
| Post title (page__title) | Montserrat | inherits MM size | 700 | colour `#343a40`, letter-spacing -0.01em |
| Footer | Montserrat | 0.82rem | 400 | colour `#888` |

**Spacing:**

- Section padding desktop: `3.5rem 0`. Mobile (≤767px): `2rem 0`.
- Navbar padding: `0.4rem 1.25rem`.
- Footer padding: `2.5rem 0`, `margin-top: 3rem`.
- Intro band padding: `2.5rem 0 2rem`.

## Sticky navbar

**File:** `_includes/masthead.html` (NEW — overrides minimal-mistakes' default)

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

**SCSS** (in `assets/css/main.scss`):

```scss
.dv-nav {
  background: #0d1b4b;
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
  font-family: $headings-font-family;
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
    font-family: $headings-font-family;
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
```

**Mobile:** No hamburger toggle. Three links collapse to a wrapped row below
the brand.

**Search:** Removed. minimal-mistakes' search modal lives in the default
masthead and is triggered from a 🔍 button there. Because we replace
`_includes/masthead.html` wholesale, the trigger button is gone and the
modal becomes unreachable. If we want search back later, add a
`<button class="search__toggle">🔍</button>` to `.dv-nav__links` and
verify the underlying modal still mounts. Out of scope for this design.

**Avatar source:** Copy
`/Users/danielvaughan/Development/git/danielvaughan.github.io/img/daniel-vaughan-en.jpg`
to `/Users/danielvaughan/Development/git/codex-blog/assets/images/daniel-vaughan-avatar.jpg`.

## Home page intro band

**File:** `index.html` (MODIFIED)

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

The `layout: home` (minimal-mistakes built-in) renders the paginated post list
automatically beneath whatever content the page itself contains. The intro
band sits above; the post list flows below.

`author_profile: false` removes the small author sidebar from the home page
only — it would compete with the navbar avatar and the intro band. The author
profile remains on individual post pages.

**SCSS:**

```scss
.dv-intro {
  background: #f8f9fa;
  border-bottom: 1px solid #e9ecef;
  padding: 2.5rem 0 2rem;
  margin-bottom: 2rem;
}
.dv-intro__inner {
  max-width: 1024px;
  margin: 0 auto;
  padding: 0 1.25rem;
}
.dv-intro h2 {
  font-family: $headings-font-family;
  font-size: 1.55rem;
  font-weight: 700;
  color: #343a40;
  margin: 0 0 0.75rem;
  padding-bottom: 0.6rem;
  border-bottom: 3px solid #0869ff;
  display: inline-block;
}
.dv-intro p {
  font-size: 1rem;
  line-height: 1.75;
  color: #323232;
  margin: 0;
  max-width: 60ch;
}
```

**Post list restyle.** Below the intro band, minimal-mistakes' `home` layout
renders post entries. They get small visual tweaks via overrides:

```scss
.archive__item-title {
  font-family: $headings-font-family;
  font-weight: 700;
  a { color: #343a40; }
  a:hover { color: #0869ff; text-decoration: none; }
}
.archive__item-excerpt {
  font-size: 0.95rem;
  color: #5a5a5a;
}
.page__meta-title { font-family: $headings-font-family; }
```

## Post page restyle

`single` layout from minimal-mistakes is unchanged structurally. Pure CSS
overrides applied via existing minimal-mistakes selectors.

```scss
// Post title
.page__title {
  font-family: $headings-font-family;
  font-weight: 700;
  color: #343a40;
  letter-spacing: -0.01em;
}

// Body headings inside post content
.page__content {
  h2, h3, h4, h5, h6 {
    font-family: $headings-font-family;
    color: #343a40;
  }
  h2 {
    font-size: 1.55rem;
    font-weight: 700;
    padding-bottom: 0.6rem;
    border-bottom: 3px solid #0869ff;
    display: inline-block;
    margin-top: 2.5rem;
  }
  h3 {
    font-size: 1.05rem;
    font-weight: 700;
    color: #0869ff;
    text-transform: uppercase;
    letter-spacing: 0.04em;
    margin-top: 2rem;
  }
  a { color: #0869ff; }
  a:hover { color: #0d1b4b; }
}

// TOC sidebar
.toc {
  .nav__title {
    background: #0d1b4b;
    color: #fff;
    font-family: $headings-font-family;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.04em;
  }
  .toc__menu a {
    font-family: $headings-font-family;
    font-size: 0.85em;
  }
  .toc__menu .active a {
    background: #f0f4ff;
    color: #0869ff;
    border-left: 3px solid #0869ff;
  }
}

// Author profile sidebar
.author__name { font-family: $headings-font-family; font-weight: 700; }
.author__bio { font-size: 0.9rem; }

// Post meta (read time, tags, share)
.page__meta, .page__taxonomy {
  font-family: $headings-font-family;
  font-size: 0.78rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #888;
}
```

**What is preserved unchanged:**
- The sketchnote image at the top of every article (rendered by the article
  body, not the layout)
- TOC sidebar position and behaviour
- Author profile sidebar
- Mermaid diagram rendering (handled by `_includes/head/custom.html`)
- Code block syntax highlighting (Rouge defaults from minimal-mistakes)
- Tables and blockquotes (MM defaults)
- Pagination, RSS feed, sitemap
- Tag archive page (`/tags/`)

## Footer

**File:** `_includes/footer.html` (NEW — overrides minimal-mistakes' default)

```html
<footer class="dv-footer">
  <div class="dv-footer__inner">
    Copyright © 2000–{{ 'now' | date: "%Y" }}. Daniel Vaughan. All rights reserved.
  </div>
</footer>
```

**SCSS:**

```scss
.dv-footer {
  border-top: 1px solid #e9ecef;
  background: #ffffff;
  padding: 2.5rem 0;
  margin-top: 3rem;
  text-align: center;
}
.dv-footer__inner {
  max-width: 1024px;
  margin: 0 auto;
  padding: 0 1.25rem;
  font-family: $headings-font-family;
  font-size: 0.82rem;
  color: #888;
}
```

Single line, centred, auto-updating year. Mirrors `_includes/footer.html`
from the main site, minus the Universal Analytics tracking script.

## Font loading

**File:** `_includes/head/custom.html` (MODIFIED)

The file already exists and currently contains only the mermaid loader.
Add Montserrat at the top. Final structure:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Montserrat:wght@400;600;700&display=swap" rel="stylesheet">

{% if site.mermaid %}
<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@{{ site.mermaid.version | default: 10 }}/dist/mermaid.esm.min.mjs';
  // ... existing mermaid converter code unchanged ...
</script>
{% endif %}
```

`Montserrat:wght@400;600;700` is the smallest set we use (regular for
fallback, 600 for nav/buttons, 700 for headings/brand).

## File inventory

**Hand-authored files in codex-blog (created or modified):**

| Path | Action | Purpose |
|---|---|---|
| `assets/css/main.scss` | Modify | Add the danielvaughan.com tokens block (~150 lines new SCSS) |
| `assets/images/daniel-vaughan-avatar.jpg` | Create | Copy from `danielvaughan.github.io/img/daniel-vaughan-en.jpg` |
| `_includes/head/custom.html` | Modify | Add Montserrat Google Fonts link |
| `_includes/masthead.html` | Create | Custom sticky navy navbar |
| `_includes/footer.html` | Create | One-line copyright stripe |
| `index.html` | Modify | Add intro band; set `author_profile: false` |
| `_data/navigation.yml` | Delete | Unused after the masthead override |

**Files NOT touched in codex-blog:**
- `_pages/tags.md` — generic selectors pick up the new styles automatically
- `_posts/*.md` — auto-synced, never edited
- `sketchnotes/articles/*` — auto-synced, never edited
- `Gemfile`, `CNAME`, `.gitignore`, `README.md`, `_config.yml` (mermaid block already present), `SYNCED-DO-NOT-EDIT.md` markers

**Files NOT touched in codex-resources:**
- The sync script and workflow are unchanged. This reskin is purely codex-blog
  hand-authored content. No re-sync is required.

## Risks and mitigations

- **CSS specificity wars with minimal-mistakes.** Minimal-mistakes has its
  own selectors with their own specificity. If a block doesn't override as
  expected, increase specificity by qualifying selectors with parent classes
  (e.g. `.layout--single .page__content h2`) rather than reaching for
  `!important`. Verify visually after each block.
- **`_includes/masthead.html` and `_includes/footer.html` overriding theme
  defaults.** Jekyll resolves includes from the local repo first, so this is
  a documented and supported pattern. Verified by inspecting minimal-mistakes
  4.26.2's default layout — `default.html` includes `masthead.html` and
  `footer.html` by name without a path prefix.
- **Mobile viewport.** The navbar collapses to a wrapped row at ≤600px. We
  test by running `bundle exec jekyll serve` and resizing the browser; if
  the layout breaks at any breakpoint we adjust the media query inline.
- **Avatar load failure.** If the avatar JPG is missing, the `<img>` shows
  alt-empty; the brand text still renders. Not catastrophic.
- **Sync script `git add -A` swallowing the new hand-authored files.** This
  is a known sync hygiene problem (documented in feedback memory). Mitigation
  for this work: commit each batch of hand-authored files before triggering
  any sync. Long-term fix is out of scope for this design.

## Acceptance criteria

The reskin is done when, on `https://codex.danielvaughan.com`:

1. A sticky dark navy navbar (`#0d1b4b`) with a 32px round avatar, "Daniel
   Vaughan" brand text in Montserrat, and three nav links (Home, Tags,
   Main site →) sits at the top of every page.
2. The home page shows a light grey intro band with "About this blog"
   (h2 with blue underline accent) and one short paragraph above the post
   list.
3. Individual post pages render with: post title in Montserrat; body h2s
   with the blue underline accent; body h3s as small uppercase blue;
   TOC sidebar with a navy header bar and blue active item; mermaid
   diagrams still rendering as SVGs; sketchnote at the top of each
   article unchanged.
4. The footer is one centred line of grey copyright text in Montserrat.
5. No console JavaScript errors on home, post, or tag pages.
6. Page loads pass `bundle exec jekyll build` without new warnings beyond
   the pre-existing Liquid template warnings on the 7 affected articles.
7. Visual comparison against `https://danielvaughan.com` shows the same
   navy navbar, the same Montserrat typeface, the same blue link colour,
   and the same h2 underline accent style.
