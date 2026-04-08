# Book Widget Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a persistent book promotion widget to `codex.danielvaughan.com` that shows a sticky floating rail on wide desktops (≥1200px) and an inline card on narrower viewports, on every article and on the home page.

**Architecture:** Configuration-driven via `_config.yml`. A single Liquid include renders both the rail and inline variants with a `placement` parameter; CSS media queries decide which is visible. No JavaScript.

**Tech Stack:** Jekyll 3, Liquid, SCSS (Bootstrap 4 + main-site overrides), Chrome DevTools MCP for verification.

**Related spec:** `docs/superpowers/specs/2026-04-08-book-widget-design.md`

**Testing note:** This is a static site with no JS test framework. "Tests" in this plan mean verification steps: building the site, grepping the compiled HTML/CSS for expected content, and visually inspecting at multiple breakpoints in a real browser via Chrome DevTools MCP. Each task ends with an explicit verification step before committing.

---

## File Structure

| File | Action | Responsibility |
|---|---|---|
| `_config.yml` | Modify (add `book:` block) | Data-only book metadata |
| `_includes/book-widget.html` | Create | Reusable widget markup, accepts `placement` param |
| `_sass/codex-blog.scss` | Modify (append section) | `.book-rail` and `.book-inline` styles |
| `_layouts/post.html` | Modify (add two includes) | Render both variants on article pages |
| `_layouts/home.html` | Modify (add two includes) | Render both variants on home + paginated listings |

Each file has one clear responsibility and can be changed independently.

---

## Task 1: Add book metadata to `_config.yml`

**Files:**
- Modify: `_config.yml` (add block after existing `author:` section, around line 46)

- [ ] **Step 1: Read current `_config.yml` to confirm insertion point**

Use the Read tool on `_config.yml`. Confirm the `author:` block ends around line 46 and `defaults:` starts at line 48. If the file has drifted, adjust the insertion point in Step 2 accordingly.

- [ ] **Step 2: Add the `book:` block**

Use Edit to insert this block immediately after the `author:` section (between `author:` end and `defaults:` start):

```yaml
# Book (promoted via _includes/book-widget.html)
book:
  title: "Codex CLI"
  subtitle: "Agentic Engineering from First Principles"
  price_text: "From $19 · pay what you want"
  cover_url: "https://d2sofvawe08yqg.cloudfront.net/codex-cli/s_hero"
  leanpub_url: "https://leanpub.com/codex-cli/"
```

- [ ] **Step 3: Verify Jekyll picks up the new config**

Run: `bundle exec jekyll build --quiet 2>&1 | tail -5`

Expected: builds cleanly with no YAML parse errors. If you see `YAML Exception` or `invalid byte sequence`, the indentation is wrong — each key under `book:` must be indented by exactly two spaces.

Additional check that the values are readable from Liquid:
Run: `bundle exec jekyll build --quiet && grep -o 'Codex CLI' _site/index.html | head -1`

Expected: matches "Codex CLI" (it's already the site title, so this just confirms the build completed). More strictly, the new `site.book.*` values aren't referenced anywhere yet, so there's nothing to grep. This step is just the clean-build check.

- [ ] **Step 4: Commit**

```bash
git add _config.yml
git commit -m "feat(config): add book metadata block for promotion widget"
```

---

## Task 2: Create the `book-widget.html` include

**Files:**
- Create: `_includes/book-widget.html`

- [ ] **Step 1: Write the include**

Use Write to create `_includes/book-widget.html` with this exact content:

```liquid
{%- comment -%}
  Book promotion widget.

  Usage:
    {% include book-widget.html placement="rail" %}
    {% include book-widget.html placement="inline" %}

  The `placement` parameter both selects the root CSS class (.book-rail or
  .book-inline) and sets the utm_medium tracking parameter so rail vs inline
  conversions can be distinguished in Leanpub's analytics.

  Book metadata is read from _config.yml under the `book:` key.
{%- endcomment -%}
{%- assign placement = include.placement -%}
{%- assign utm_url = site.book.leanpub_url | append: "?utm_source=codex-blog&utm_medium=" | append: placement | append: "&utm_campaign=book-widget" -%}
<aside class="book-{{ placement }}" aria-label="Buy the book: {{ site.book.title | escape }}">
  <a class="book-{{ placement }}__cover" href="{{ utm_url }}" tabindex="-1" rel="noopener" target="_blank">
    <img src="{{ site.book.cover_url }}" alt="Cover of the book {{ site.book.title | escape }} by {{ site.author.name | escape }}">
  </a>
  <div class="book-{{ placement }}__body">
    <p class="book-{{ placement }}__title">{{ site.book.title | escape }}</p>
    <p class="book-{{ placement }}__subtitle">{{ site.book.subtitle | escape }}</p>
    <p class="book-{{ placement }}__price">{{ site.book.price_text | escape }}</p>
    <a class="book-{{ placement }}__cta" href="{{ utm_url }}" rel="noopener" target="_blank">
      Buy on Leanpub <span aria-hidden="true">→</span>
    </a>
  </div>
</aside>
```

- [ ] **Step 2: Verify the include file is syntactically valid Liquid**

Run: `bundle exec jekyll build --quiet 2>&1 | tail -5`

Expected: build still succeeds (the include exists but is not yet referenced, so nothing changes in `_site/`). If you see `Liquid Exception` or `Liquid syntax error`, re-read the include file and check for mismatched `{% %}` tags.

- [ ] **Step 3: Commit**

```bash
git add _includes/book-widget.html
git commit -m "feat(includes): add book-widget partial with placement parameter"
```

---

## Task 3: Add book widget SCSS

**Files:**
- Modify: `_sass/codex-blog.scss` (append new section at end of file)

- [ ] **Step 1: Read the end of the SCSS file to confirm insertion point**

Use Read on `_sass/codex-blog.scss`. Find the last line of the file. Append the new section after the existing last rule (do NOT replace any existing content).

- [ ] **Step 2: Append the book widget styles**

Use Edit to append this block at the END of `_sass/codex-blog.scss`:

```scss

// ---------- Book widget ----------
// Two variants share one source file. The rail is a fixed-position floating
// element shown only at ≥xl breakpoint (1200px). The inline card is a
// flex-row card shown inside the article at smaller viewports. Both pull
// content from the same _includes/book-widget.html using a `placement` param.
//
// IMPORTANT: The `left` calc assumes the main-site override pins the xl
// container at 720px (see _main-site/_sass/_variables.scss). If that
// override changes, this math breaks.

// --- Desktop floating rail (≥1200px) ---
.book-rail {
  display: none;
  @include media-breakpoint-up(xl) {
    display: block;
    position: fixed;
    top: 90px;                                 // below sticky navbar
    left: calc((100vw - 720px) / 2 - 170px);  // 150px rail + 20px gap from container
    width: 150px;
    z-index: 10;
    padding: 1rem;
    background: #fff;
    border: 1px solid $gray-200;
    border-radius: $border-radius;
    box-shadow: 0 4px 14px rgba(0, 0, 0, 0.08);
    text-align: center;
  }
  &__cover img {
    width: 100%;
    height: auto;
    border-radius: 3px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.18);
    display: block;
  }
  &__body { margin-top: 0.75rem; }
  &__title {
    font-family: $headings-font-family;
    font-weight: 700;
    font-size: 1rem;
    color: $gray-800;
    line-height: 1.25;
    margin: 0 0 0.15rem;
  }
  &__subtitle {
    font-size: 0.75rem;
    color: $gray-700;
    line-height: 1.3;
    margin: 0 0 0.5rem;
  }
  &__price {
    font-size: 0.75rem;
    color: $gray-700;
    margin: 0 0 0.75rem;
  }
  &__cta {
    display: block;
    background: $primary;
    color: #fff;
    padding: 0.5rem 0.75rem;
    border-radius: $border-radius-sm;
    text-decoration: none;
    font-family: $headings-font-family;
    font-size: 0.8rem;
    font-weight: 700;
    &:hover,
    &:focus {
      background: darken($primary, 10%);
      color: #fff;
      text-decoration: none;
    }
  }
}

// --- Mobile / narrow-desktop inline card (<1200px) ---
.book-inline {
  display: flex;
  flex-direction: row;
  align-items: center;
  gap: 1rem;
  background: #fff;
  border: 1px solid $gray-200;
  border-radius: $border-radius;
  padding: 1rem;
  margin: 0 0 2rem;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
  @include media-breakpoint-up(xl) {
    display: none;                             // rail takes over ≥1200px
  }
  &__cover {
    flex: 0 0 80px;
    img {
      width: 100%;
      height: auto;
      border-radius: 3px;
      box-shadow: 0 2px 8px rgba(0, 0, 0, 0.18);
      display: block;
    }
  }
  &__body {
    flex: 1;
    min-width: 0;
  }
  &__title {
    font-family: $headings-font-family;
    font-weight: 700;
    font-size: 1.05rem;
    color: $gray-800;
    line-height: 1.25;
    margin: 0 0 0.15rem;
  }
  &__subtitle {
    font-size: 0.85rem;
    color: $gray-700;
    line-height: 1.3;
    margin: 0 0 0.35rem;
  }
  &__price {
    font-size: 0.8rem;
    color: $gray-700;
    margin: 0 0 0.6rem;
  }
  &__cta {
    display: inline-block;
    background: $primary;
    color: #fff;
    padding: 0.45rem 0.85rem;
    border-radius: $border-radius-sm;
    text-decoration: none;
    font-family: $headings-font-family;
    font-size: 0.8rem;
    font-weight: 700;
    &:hover,
    &:focus {
      background: darken($primary, 10%);
      color: #fff;
      text-decoration: none;
    }
  }
}
```

- [ ] **Step 3: Run Jekyll build and confirm SCSS compiles**

Run: `bundle exec jekyll build --quiet 2>&1 | tail -10`

Expected: builds cleanly. If you see `Error: ... SASS`, re-read your appended block for missing semicolons or unclosed braces.

- [ ] **Step 4: Grep the compiled CSS for the new rules**

Run the Grep tool on `_site/assets/css/main.css` for `book-rail` — expect multiple matches including `.book-rail { display: none }` and the `@media (min-width: 1200px)` rule containing `position: fixed`. Also grep for `.book-inline` and confirm the base rule is `display: flex`.

- [ ] **Step 5: Commit**

```bash
git add _sass/codex-blog.scss
git commit -m "feat(style): add book-rail and book-inline widget styles"
```

---

## Task 4: Include the widget in the post layout

**Files:**
- Modify: `_layouts/post.html`

- [ ] **Step 1: Read current `_layouts/post.html`**

Use the Read tool. Confirm the structure: `<article class="post">` with `<header class="post-header">` ending at line ~20, then `<nav class="post-toc">`, then `<section class="post-body">`.

- [ ] **Step 2: Add both includes inside `<article class="post">` but outside `<header>`**

The rail goes first (outside any flow container — it's position-fixed anyway). The inline card goes immediately after the `</header>` and before the `<nav class="post-toc">`, so on narrow viewports it sits right after the post header and before the TOC.

Use Edit to make this change:

Old string (the entire existing post.html article block):
```html
<article class="post">
  <header class="post-header">
    <h1 class="post-title">{{ page.title | escape }}</h1>
    <p class="post-meta">
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
```

New string:
```html
{% include book-widget.html placement="rail" %}
<article class="post">
  <header class="post-header">
    <h1 class="post-title">{{ page.title | escape }}</h1>
    <p class="post-meta">
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

  {% include book-widget.html placement="inline" %}

  <nav class="post-toc" aria-label="On this page" hidden>
```

- [ ] **Step 3: Build and confirm the widget markup lands in a rendered post**

Run: `bundle exec jekyll build --quiet`

Then use Grep on `_site` for `book-rail` in `.html` files, expecting matches in every post under `_site/2026/` and similar. Example:

Run: Grep with pattern `book-rail` path `_site` glob `**/*.html`, output_mode `files_with_matches`. Expect at least 100+ files (one per post).

Also Grep one specific post to confirm both placements are present. Pick any file from the Grep results, e.g. `_site/2026/03/26/what-is-codex-cli/index.html`, and Grep for `book-rail` and then for `book-inline` in that single file. Both should match.

- [ ] **Step 4: Confirm the UTM parameters are built correctly**

Run: Grep with pattern `utm_medium=rail` path `_site` glob `**/*.html`, output_mode `files_with_matches`, head_limit 1.

Expected: at least one match. Read the matched file around the match to confirm the full URL is `https://leanpub.com/codex-cli/?utm_source=codex-blog&utm_medium=rail&utm_campaign=book-widget`.

Do the same for `utm_medium=inline`.

- [ ] **Step 5: Commit**

```bash
git add _layouts/post.html
git commit -m "feat(layout): render book widget (rail + inline) on posts"
```

---

## Task 5: Include the widget in the home layout

**Files:**
- Modify: `_layouts/home.html`

- [ ] **Step 1: Read current `_layouts/home.html`**

Use Read. Confirm the two `<section class="content-section">` blocks: the intro `bg-alt` section (shown only on `paginator.page == 1`) and the posts listing `bg-white` section.

- [ ] **Step 2: Add both includes**

The rail should sit at the top of the layout (it's fixed position, placement in markup doesn't affect visual position). The inline card should sit immediately after the "About this blog" intro band if present, or at the top of the posts list if not.

Use Edit to make this change:

Old string:
```html
<h1 class="sr-only">{{ page.title | default: site.title }}</h1>

{%- if paginator.page == 1 -%}
<section class="content-section bg-alt">
  <div class="container">
    <h2>About this blog</h2>
    <p>Notes on agentic software engineering, Codex CLI, and how AI is changing the way we build software. Published as I write them.</p>
  </div>
</section>
{%- endif -%}

<section class="content-section bg-white">
  <div class="container">
    {%- for post in paginator.posts -%}
```

New string:
```html
<h1 class="sr-only">{{ page.title | default: site.title }}</h1>

{% include book-widget.html placement="rail" %}

{%- if paginator.page == 1 -%}
<section class="content-section bg-alt">
  <div class="container">
    <h2>About this blog</h2>
    <p>Notes on agentic software engineering, Codex CLI, and how AI is changing the way we build software. Published as I write them.</p>
  </div>
</section>
{%- endif -%}

<section class="content-section bg-white">
  <div class="container">
    {% include book-widget.html placement="inline" %}
    {%- for post in paginator.posts -%}
```

- [ ] **Step 3: Build and verify the widget appears on the home page and paginated listings**

Run: `bundle exec jekyll build --quiet`

Then:

- Grep for `book-rail` in `_site/index.html` — expect at least one match (the rail is there once).
- Grep for `book-inline` in `_site/index.html` — expect at least one match.
- Grep for `book-rail` in `_site/page2/index.html` — expect at least one match (paginated listings also include the widget).

- [ ] **Step 4: Confirm the widget does NOT appear on the tags page**

Run: Grep with pattern `book-rail` path `_site/tags` output_mode `files_with_matches`.

Expected: no matches. If there are matches, the widget leaked through a shared layout — investigate before proceeding.

- [ ] **Step 5: Commit**

```bash
git add _layouts/home.html
git commit -m "feat(layout): render book widget on home and paginated listings"
```

---

## Task 6: Visual verification with Chrome DevTools MCP

**Files:**
- None (verification only)

This task verifies the widget renders correctly at multiple viewport widths against a locally-served Jekyll site. It catches issues the compiled HTML/CSS grep can't: the calc positioning, the actual rendered widths, the contrast of real text on real backgrounds, and mobile tap targets.

- [ ] **Step 1: Start the local Jekyll server in the background**

Run: `bundle exec jekyll serve --quiet --port 4000` with `run_in_background: true`.

Expected: server starts and listens on http://localhost:4000. Confirm by navigating to the URL in the next step.

- [ ] **Step 2: Navigate to a sample article at 1440px width**

Use `mcp__plugin_chrome-devtools-mcp_chrome-devtools__resize_page` to set width=1440, height=900.

Then `mcp__plugin_chrome-devtools-mcp_chrome-devtools__navigate_page` to `http://localhost:4000/2026/03/26/what-is-codex-cli/` (or any other article URL — pick one from `_site/2026/` listing).

- [ ] **Step 3: Confirm the floating rail is visible and correctly positioned**

Use `mcp__plugin_chrome-devtools-mcp_chrome-devtools__evaluate_script`:

```js
() => {
  const rail = document.querySelector('.book-rail');
  if (!rail) return { error: 'rail not found' };
  const cs = getComputedStyle(rail);
  const rect = rail.getBoundingClientRect();
  const container = document.querySelector('.container');
  const containerRect = container ? container.getBoundingClientRect() : null;
  return {
    display: cs.display,
    position: cs.position,
    top: cs.top,
    left: cs.left,
    width: cs.width,
    rail: { left: rect.left, right: rect.right, width: rect.width },
    containerLeft: containerRect ? containerRect.left : null,
    gap: containerRect ? containerRect.left - rect.right : null,
  };
}
```

Expected:
- `display`: `block`
- `position`: `fixed`
- `width`: `150px`
- `rail.left`: ≥0 (on-screen)
- `gap`: approximately 20 (the rail's right edge is ~20px left of the article container's left edge)

If `gap` is significantly off (e.g. <10 or >40), the calc math is wrong; investigate the container width at 1440px.

- [ ] **Step 4: Confirm the inline card is hidden at 1440px**

Continue the same `evaluate_script` session:

```js
() => {
  const inline = document.querySelector('.book-inline');
  return inline ? getComputedStyle(inline).display : 'not found';
}
```

Expected: `none`.

- [ ] **Step 5: Take a screenshot at 1440px**

Use `take_screenshot` with `filePath: /tmp/book-widget-1440.png`. Visually confirm:
- Rail sits on the left with cover, title, subtitle, price, CTA button
- Article content is unchanged on the right
- No layout shift in the article

- [ ] **Step 6: Resize to exactly 1200px and re-verify**

Use `resize_page` to 1200×900. Re-run the `evaluate_script` from Step 3. Expected: rail still visible, `gap` now approximately 70 (more room to spare at the exact breakpoint), and the inline card still `display: none`.

- [ ] **Step 7: Resize to 1199px and confirm the inline card takes over**

Use `resize_page` to 1199×900.

Run:

```js
() => {
  const rail = document.querySelector('.book-rail');
  const inline = document.querySelector('.book-inline');
  return {
    rail: rail ? getComputedStyle(rail).display : 'not found',
    inline: inline ? getComputedStyle(inline).display : 'not found',
  };
}
```

Expected: `rail: none`, `inline: flex`.

Take a screenshot to `/tmp/book-widget-1199.png`. Confirm the inline card sits at the top of the article, below the post header.

- [ ] **Step 8: Resize to 375px and confirm mobile layout**

Use `resize_page` to 375×812 (iPhone X). Re-run the same display-check script. Expected: `rail: none`, `inline: flex`.

Take a screenshot to `/tmp/book-widget-375.png`. Confirm:
- Cover image is ~80px wide on the left
- Title, subtitle, price, and CTA are stacked on the right
- CTA button is at least 44×44px tappable (visually inspect)

- [ ] **Step 9: Navigate to the home page at 1440px and verify the rail**

Use `resize_page` to 1440×900, then `navigate_page` to `http://localhost:4000/`. Re-run Step 3's script. Expected: rail visible, inline hidden.

- [ ] **Step 10: Navigate to the tags page and confirm neither variant appears**

Navigate to `http://localhost:4000/tags/`. Run:

```js
() => ({
  rail: document.querySelector('.book-rail') ? 'found' : 'absent',
  inline: document.querySelector('.book-inline') ? 'found' : 'absent',
})
```

Expected: both `absent`. If either is `found`, the widget is leaking into scope it shouldn't be in — stop and investigate.

- [ ] **Step 11: Verify the CTA link has the correct UTM parameters**

On the article page (navigate back if needed), run:

```js
() => {
  const railCta = document.querySelector('.book-rail .book-rail__cta');
  const inlineCta = document.querySelector('.book-inline .book-inline__cta');
  return {
    rail: railCta ? railCta.href : null,
    inline: inlineCta ? inlineCta.href : null,
  };
}
```

Expected:
- `rail`: `https://leanpub.com/codex-cli/?utm_source=codex-blog&utm_medium=rail&utm_campaign=book-widget`
- `inline`: `https://leanpub.com/codex-cli/?utm_source=codex-blog&utm_medium=inline&utm_campaign=book-widget`

Note: at 1440px the inline card has `display: none` but it's still in the DOM and queryable.

- [ ] **Step 12: Contrast check on the CTA button**

Run:

```js
() => {
  const cta = document.querySelector('.book-rail .book-rail__cta');
  if (!cta) return null;
  const cs = getComputedStyle(cta);
  return { color: cs.color, background: cs.backgroundColor };
}
```

Expected: `color: rgb(255, 255, 255)` (white) on `background: rgb(8, 105, 255)` (the `$primary` blue). This is the same contrast ratio as the pagination buttons (~5.17:1, AA pass).

- [ ] **Step 13: Kill the Jekyll server**

Use KillShell on the background shell you started in Step 1.

- [ ] **Step 14: Commit screenshots to the brainstorm directory for reference** *(optional)*

The screenshots in `/tmp` are not committed. If you want to keep them, copy them into `.superpowers/brainstorm/` (which is gitignored) for local reference, or delete them. No commit needed for this task since no source files changed.

---

## Task 7: Final verification — full article rendering

**Files:**
- None (integration test)

One last sanity pass: make sure no existing pages are broken by the new widget.

- [ ] **Step 1: Build the site cleanly**

Run: `rm -rf _site && bundle exec jekyll build --quiet 2>&1 | tail -20`

Expected: clean build, no errors, no deprecation warnings that weren't there before.

- [ ] **Step 2: Check the build produced the expected number of files**

Run: Glob pattern `_site/2026/**/*.html`

Expected: the same number of files as before (no posts dropped). If you want a number: count the files in `_posts/` and compare; they should match 1:1.

- [ ] **Step 3: Spot-check three random articles for widget presence**

Run Grep with pattern `book-rail` path `_site/2026` output_mode `files_with_matches`. Pick three random results (e.g. `head -1`, `sed -n '50p'`, `tail -1` equivalents — use head_limit and offset on Grep). For each, also confirm `book-inline` is present.

- [ ] **Step 4: Confirm the pagination fix from earlier is still intact**

This protects against the plan accidentally reverting the earlier contrast fix.

Run: Grep with pattern `pagination .page-link` in `_site/assets/css/main.css`.

Expected: no match for `.pagination .page-link { color: #0869ff }` (the broken rule should still be gone). Also confirm the `.pagination .page-link:hover` background is `#d71921` (the darkened pink from the earlier fix).

(No commit for Task 7 — verification only, no source changes.)

---

## Task 8: Push

**Files:**
- None

- [ ] **Step 1: Review the local commit log**

Run: `git log --oneline origin/main..HEAD`

Expected: 5 new commits, one per Task 1–5. Task 6 and 7 were verification-only.

- [ ] **Step 2: Fetch and check for upstream changes**

Run: `git fetch origin main && git log --oneline HEAD..origin/main`

If there are sync-bot commits upstream, rebase on top of them:

```bash
git rebase origin/main
```

Expected: clean rebase. The sync bot doesn't touch `_sass/`, `_includes/`, `_layouts/`, or `_config.yml`, so there should be no conflicts.

- [ ] **Step 3: Push**

Run: `git push origin main`

Expected: push succeeds. If rejected, fetch and rebase again, then re-push.

- [ ] **Step 4: Confirm the deploy completes**

The GitHub Pages deploy will run automatically. Optionally, navigate to `https://codex.danielvaughan.com/` in a real browser after ~1-2 minutes and verify the rail appears on a wide desktop and the inline card on mobile. (This is a smoke test; the DevTools MCP verification in Task 6 is the authoritative check.)

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| Book metadata in `_config.yml` | Task 1 |
| Reusable `_includes/book-widget.html` with `placement` param | Task 2 |
| `.book-rail` SCSS (≥1200px, fixed position) | Task 3 |
| `.book-inline` SCSS (<1200px, flex row) | Task 3 |
| `display: none` hides one variant per viewport (a11y) | Task 3 (both variants gated) + Task 6 Step 4/7 |
| Widget rendered twice on post layout | Task 4 |
| Widget rendered twice on home layout | Task 5 |
| No widget on tags/pages layout | Task 5 Step 4 + Task 6 Step 10 |
| UTM parameters with `utm_medium=rail`/`utm_medium=inline` | Task 2 (include) + Task 4 Step 4 + Task 6 Step 11 |
| Accessibility: `<aside aria-label>`, `alt`, `tabindex="-1"` on cover, `rel="noopener"` | Task 2 (include markup) |
| CTA contrast check (WCAG AA) | Task 6 Step 12 |
| Visual regression at multiple breakpoints | Task 6 Steps 2–10 |
| Confirm container-width assumption (720px at xl) | Task 6 Step 3 (reads gap from real DOM) |

All spec requirements mapped.

**Placeholder scan:** No TODOs, no "handle errors appropriately", all code shown in full, all commands explicit.

**Type/name consistency:** CSS class names use BEM-style (`.book-rail__cover`, `.book-rail__cta`, etc.) consistently across the include (Task 2) and the SCSS (Task 3). The `placement` variable is named identically in the include and in the instructions.
