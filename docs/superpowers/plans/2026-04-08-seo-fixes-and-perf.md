# SEO Fixes & Perf Wins Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the broken tag anchor links across the site, differentiate paginated-page meta descriptions, compress the SCSS output, and drop ~135KB of unused jQuery + Bootstrap JS per page by replacing them with a 15-line vanilla navbar toggle.

**Architecture:** Four small, independent fixes wrapped in one PR. Each is self-contained: one file modification, straightforward verification, immediate commit. No structural changes, no CSS refactoring.

**Tech Stack:** Jekyll 3 (GitHub Pages legacy builder), Liquid, SCSS (Bootstrap 4 imported via main-site submodule).

**Related plan:** `docs/superpowers/plans/2026-04-08-seo-pr3-internal-linking.md` — sibling PR. This new plan is a set of corrections/optimisations informed by the SEO-negative audit after PR 3 landed.

**Testing note:** Static site, no unit tests. Verification = `bundle exec jekyll build --quiet` + Grep on `_site/**` + Chrome DevTools MCP for the navbar interaction check.

---

## File Structure

| File | Action | Responsibility |
|---|---|---|
| `_layouts/post.html` | Modify | Fix tag pill `href` from `/tags/#{slug}` to `/tags/#tag-{slug}` |
| `_layouts/home.html` | Modify | Fix tag pill `href` (same bug) AND differentiate paginated-page meta description |
| `_config.yml` | Modify | Add `sass.style: compressed` so jekyll-sass-converter emits minified CSS |
| `_includes/head.html` | Modify | Remove the two CDN `<script>` tags for jQuery and Bootstrap JS |
| `_includes/head/custom.html` | Modify | Append a small vanilla JS navbar-toggle handler |

Each file has one responsibility change per task. No task touches more than two files.

---

## Task 1: Fix tag anchor links in post layout

**Files:**
- Modify: `/Users/danielvaughan/Development/git/codex-blog/_layouts/post.html` (one line, the tag pill anchor)

**Context:** The current expression emits `/tags/#{tag | slugify}` but the tags page sections use `id="tag-{tag | slugify}"`. Without the `tag-` prefix in the anchor, clicking a pill loads `/tags/` successfully but scrolls to nothing (no matching element). The fix: add `#tag-` to the href template instead of `#`.

- [ ] **Step 1: Read `_layouts/post.html`**

Use Read. Find the line that builds the tag `<a>` inside `<div class="post-tags">`. Current:
```html
<a href="{{ '/tags/#' | append: tag | slugify | relative_url }}" class="badge">{{ tag }}</a>
```

- [ ] **Step 2: Fix the href**

Use Edit with this change.

Old string:
```
      <a href="{{ '/tags/#' | append: tag | slugify | relative_url }}" class="badge">{{ tag }}</a>
```

New string:
```
      <a href="{{ '/tags/#tag-' | append: tag | slugify | relative_url }}" class="badge">{{ tag }}</a>
```

(Yes, that's the entire fix — add `tag-` after the `#` inside the literal string that gets appended to `/tags/`.)

- [ ] **Step 3: Build and verify**

Run: `cd /Users/danielvaughan/Development/git/codex-blog && bundle exec jekyll build --quiet 2>&1 | tail -5`

Expected: clean build (only the faraday-retry warning).

Use Grep tool on `_site/2026/03/27/what-is-codex-cli/index.html`:
- Pattern `href="/tags/#tag-` → expect multiple matches (one per tag on that article)
- Pattern `href="/tags/#codex-cli"` (the OLD broken form) → expect NO match

- [ ] **Step 4: Verify the target anchor exists on the tags page**

Grep `_site/tags/index.html` for an anchor the post links to. Pick a tag you know exists (e.g. `codex-cli`). Use Grep pattern `id="tag-codex-cli"` — expected: exactly one match.

This confirms the fix is correct: the post links to `#tag-codex-cli` and the tags page has an element with `id="tag-codex-cli"`.

- [ ] **Step 5: Commit**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git add _layouts/post.html
git commit -m "fix(layout): correct tag anchor link to include tag- prefix"
```

Only stage `_layouts/post.html`. Do NOT use `git add -A`.

---

## Task 2: Fix tag anchor links in home layout + differentiate paginated descriptions

**Files:**
- Modify: `/Users/danielvaughan/Development/git/codex-blog/_layouts/home.html` (tag pill fix + pagination description)

**Context:** Two fixes in one file. (a) Same tag anchor bug as Task 1 but in the post-card on the home layout. (b) Paginated pages (`/`, `/page2/`, …) currently all share the same `<meta description>` which signals near-duplicate content to Google.

For (b), we'll set `page.description` via a conditional at the top of the layout so `jekyll-seo-tag` picks it up. When `paginator.page > 1`, we'll append " (page N of M)" to the site description.

- [ ] **Step 1: Read `_layouts/home.html`**

Use Read on `/Users/danielvaughan/Development/git/codex-blog/_layouts/home.html`. Find:
1. The tag pill `<a>` line near the top inside the post card loop
2. The first line after `---` layout front matter (where the page-level description override should go)

- [ ] **Step 2: Fix the tag anchor**

Use Edit with this change.

Old string:
```
        <a href="{{ '/tags/#' | append: tag | slugify | relative_url }}" class="badge">{{ tag }}</a>
```

New string:
```
        <a href="{{ '/tags/#tag-' | append: tag | slugify | relative_url }}" class="badge">{{ tag }}</a>
```

- [ ] **Step 3: Add paginated-description differentiation**

Use Edit again.

Old string:
```
{%- comment -%}
  Visually-hidden h1 for screen readers and SEO. The intro band's h2 is the
  visible primary heading; an h1 is required for proper document outline.
{%- endcomment -%}
<h1 class="sr-only">{{ page.title | default: site.title }}</h1>
```

New string:
```
{%- comment -%}
  Visually-hidden h1 for screen readers and SEO. The intro band's h2 is the
  visible primary heading; an h1 is required for proper document outline.
{%- endcomment -%}
<h1 class="sr-only">{{ page.title | default: site.title }}</h1>

{%- comment -%}
  Differentiate paginated-page meta descriptions so Google doesn't treat
  /page2/, /page3/, etc. as duplicates of the home page. We override the
  description at render time via a capture, then assign it into page so
  jekyll-seo-tag picks it up on its next emission of <meta description>.
{%- endcomment -%}
{%- if paginator.page > 1 -%}
  {%- capture paginated_description -%}{{ site.description }} — page {{ paginator.page }} of {{ paginator.total_pages }}{%- endcapture -%}
  {%- assign page_description_override = paginated_description -%}
{%- endif -%}
```

Note on the mechanism: Liquid can't directly mutate `page.description`, but `jekyll-seo-tag` checks `page.description` → `page.excerpt` → `site.description` in that order. We need a different approach.

Actually, the simplest working approach: set `description` via front matter with a `paginate_description` fallback. Since paginated pages are generated by jekyll-paginate from the same `home.html` file, we can't add per-page front matter.

The REAL working approach: put the `description` meta tag inline in `head.html` or in the layout, bypassing jekyll-seo-tag for paginated pages. But that creates a duplicate tag.

Simpler pivot: **change home.html to use jekyll-seo-tag's behaviour where `page.description` (if set) wins over `site.description`.** We do this by adding a conditional `<meta name="description">` in `home.html` that overrides jekyll-seo-tag's output. BUT since jekyll-seo-tag emits the tag from `<head>`, our override in `home.html` (which is inside `<body>`) won't work.

**Actual working approach that requires minimal plumbing:** add the description override to `_includes/head.html` directly as a conditional that runs BEFORE `{% seo %}`. jekyll-seo-tag emits `<meta name="description">` based on `page.description`; if we assign a Liquid variable that jekyll-seo-tag picks up, the issue is jekyll-seo-tag reads the `page` object not a local variable.

OK given this plan is getting complex, let's take a pragmatic shortcut: **manually emit a `<meta name="description">` tag in head.html BEFORE jekyll-seo-tag's `{% seo %}` call, conditional on being a paginated page. Then the FIRST meta description in the document has the differentiated text, and Google will use that (parsers use the first tag). jekyll-seo-tag's default tag still renders after, as a fallback, but the first one wins.**

Revert the home.html description edit (don't make the change described above in this task). Instead, move the paginated-description fix into a new Task 2a that modifies `_includes/head.html`. See below.

For Task 2 proper, ONLY do the tag anchor fix in home.html. The paginated description fix moves to Task 2a.

- [ ] **Step 3 (REVISED): Don't add the paginated-description block to home.html**

Skip the Step 3 edit above. home.html gets ONLY the tag anchor fix (Step 2).

- [ ] **Step 4: Build and verify the tag anchor fix**

Run: `bundle exec jekyll build --quiet 2>&1 | tail -5`

Expected: clean build.

Grep on `_site/index.html`:
- Pattern `href="/tags/#tag-` → expect matches (the new form)
- Pattern `href="/tags/#codex-cli"` → expect NO match (the old broken form should be gone)

- [ ] **Step 5: Commit**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git add _layouts/home.html
git commit -m "fix(layout): correct tag anchor link to include tag- prefix (home)"
```

---

## Task 2a: Paginated-page meta description differentiation

**Files:**
- Modify: `/Users/danielvaughan/Development/git/codex-blog/_includes/head.html`

**Context:** Add a conditional `<meta name="description">` BEFORE `{% seo %}` in `head.html` that emits on paginated home pages (when `paginator.page > 1`). Because our custom tag is the FIRST `<meta name="description">` in the document, most search engines use it; `jekyll-seo-tag`'s fallback still emits after. For non-paginated pages (`paginator.page == 1` or no paginator) nothing is emitted and `jekyll-seo-tag`'s tag is the only one.

- [ ] **Step 1: Read `_includes/head.html`**

Use Read. Locate the `{%- include head/og-image.html -%}` line and the `{%- seo -%}` line that follows it.

- [ ] **Step 2: Add paginated-description override before `{%- seo -%}`**

Use Edit:

Old string:
```
    {%- include head/og-image.html -%}
    {%- seo -%}
```

New string:
```
    {%- include head/og-image.html -%}
    {%- if paginator and paginator.page and paginator.page > 1 -%}
    <meta name="description" content="{{ site.description | escape }} — page {{ paginator.page }} of {{ paginator.total_pages }}">
    {%- endif -%}
    {%- seo -%}
```

- [ ] **Step 3: Build and verify**

Run: `bundle exec jekyll build --quiet 2>&1 | tail -5`

Expected: clean build.

Verify the override appears on paginated pages:
- Use Grep on `_site/page2/index.html` with pattern `page 2 of` → expect match showing `content="... — page 2 of 19"` in a `<meta name="description">` tag.
- Use Grep on `_site/page3/index.html` with pattern `page 3 of` → expect match.

Verify the override does NOT appear on the home page (first page):
- Grep `_site/index.html` with pattern `page 1 of` → expect NO match.
- Grep `_site/index.html` with pattern `— page` → expect NO match.

Verify no duplicate descriptions on articles (they don't have a paginator):
- Grep `_site/2026/03/27/what-is-codex-cli/index.html` with pattern `— page` → expect NO match.

- [ ] **Step 4: Commit**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git add _includes/head.html
git commit -m "fix(head): differentiate paginated home description with page number"
```

---

## Task 3: Compress SCSS output via _config.yml

**Files:**
- Modify: `/Users/danielvaughan/Development/git/codex-blog/_config.yml` (add `sass.style: compressed` under the existing `sass:` block)

**Context:** Jekyll's `jekyll-sass-converter` accepts a `style:` option under `sass:` in `_config.yml`. Setting it to `compressed` minifies the output CSS (strips comments, whitespace, shortens hex values, etc.). Current main.css is ~187KB unminified; expect ~90-100KB after compression. Zero layout risk because the OUTPUT is just the same CSS minified.

- [ ] **Step 1: Read current `_config.yml` `sass:` block**

Use Read. Find the existing `sass:` block. It currently looks like:

```yaml
sass:
  load_paths:
    - _sass
    - _main-site/_sass
```

- [ ] **Step 2: Add `style: compressed`**

Use Edit:

Old string:
```yaml
sass:
  load_paths:
    - _sass
    - _main-site/_sass
```

New string:
```yaml
sass:
  style: compressed
  load_paths:
    - _sass
    - _main-site/_sass
```

- [ ] **Step 3: Build and measure the CSS size drop**

Run: `cd /Users/danielvaughan/Development/git/codex-blog && bundle exec jekyll build --quiet 2>&1 | tail -5`

Expected: clean build.

Check the new size:
```bash
wc -c _site/assets/css/main.css
```

Record the new size and compute the % reduction vs the pre-fix baseline of ~191,608 bytes. Expected: 40-60% smaller. A typical compressed main.css for this site should be around 90,000-115,000 bytes. If the result is still above 150,000 bytes, something went wrong — the compression didn't apply.

- [ ] **Step 4: Spot-check that the CSS still loads and the site still looks OK**

No rendering check here — trusted because SCSS compression is a lossless operation. Task 5 (verification) will do a visual spot-check anyway.

But DO grep the compiled CSS for the pagination contrast fix from earlier to make sure it survived:

Use Grep on `_site/assets/css/main.css` with pattern `\.pagination \.page-item\.disabled` → expected: match (the fix from the original pagination readability commit).

And grep for `\.book-rail` → expected: match.

And grep for `\.related-posts` → expected: match.

These confirm no existing rules were dropped by the minifier.

- [ ] **Step 5: Commit**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git add _config.yml
git commit -m "perf(sass): enable compressed CSS output (-50% size)"
```

---

## Task 4: Replace jQuery + Bootstrap JS bundle with vanilla navbar toggle

**Files:**
- Modify: `/Users/danielvaughan/Development/git/codex-blog/_includes/head.html` (remove the two CDN script tags)
- Modify: `/Users/danielvaughan/Development/git/codex-blog/_includes/head/custom.html` (append a small vanilla navbar-toggle handler)

**Context:** The site currently loads jQuery 3.5.1 slim (~28KB gzipped) and Bootstrap 4.6.2 JS bundle (~58KB gzipped) from CDNs on every page. The only feature in use is the navbar collapse toggle on mobile — when the hamburger button is tapped, the nav collapses/expands. Everything else Bootstrap JS provides (modals, dropdowns, tooltips, popovers) is unused. Replacing the two scripts with a ~15-line vanilla JS handler saves ~86KB gzipped per page with zero functional regression.

- [ ] **Step 1: Remove the two CDN script tags from `_includes/head.html`**

Read `_includes/head.html` first to see the current content.

Use Edit:

Old string:
```
    <script src="https://code.jquery.com/jquery-3.5.1.slim.min.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@4.6.2/dist/js/bootstrap.bundle.min.js" crossorigin="anonymous"></script>
</head>
```

New string:
```
</head>
```

(That's removing both `<script>` lines and their preceding whitespace, leaving the `</head>` closing tag.)

- [ ] **Step 2: Append the vanilla navbar-toggle handler to `_includes/head/custom.html`**

Read the current `_includes/head/custom.html` to find the end of the file. Append the new code AFTER the existing content (do not replace any).

The existing file has this structure (verify by reading):
```html
<link rel="preconnect" ...>
...
<script>
  document.addEventListener('DOMContentLoaded', function () {
    // existing TOC + h1 hider logic
    ...
  });
</script>

{% if site.mermaid %}
<script type="module">
  ...
</script>
{% endif %}
```

Use Edit to add a new `<script>` block AFTER the existing inline `</script>` tag that contains the TOC logic, but BEFORE the `{% if site.mermaid %}` block.

Actually, to avoid sequencing issues with the old_string, the cleanest approach is to INSERT the new script INSIDE the existing DOMContentLoaded handler, alongside the TOC logic. That way we only add new logic to an existing listener, not a new listener.

Read the custom.html file and locate the closing brace of the existing DOMContentLoaded listener. It ends with something like:
```javascript
    nav.removeAttribute('hidden');
  });
</script>
```

Use Edit with this change:

Old string:
```
    nav.removeAttribute('hidden');
  });
</script>
```

New string:
```
    nav.removeAttribute('hidden');

    // 3) Navbar toggler — replaces Bootstrap 4 JS collapse plugin. Tapping the
    //    hamburger button toggles the `show` class on the matching target.
    //    Bootstrap 4's .collapse.show { display: block; } (from the SCSS
    //    pipeline) handles the visual state; we only handle the click.
    var toggler = document.querySelector('.navbar-toggler');
    if (toggler) {
      var targetSel = toggler.getAttribute('data-target');
      var target = targetSel ? document.querySelector(targetSel) : null;
      if (target) {
        toggler.addEventListener('click', function (e) {
          e.preventDefault();
          var isShown = target.classList.toggle('show');
          toggler.setAttribute('aria-expanded', isShown ? 'true' : 'false');
        });
      }
    }
  });
</script>
```

This appends a third numbered block inside the same DOMContentLoaded listener. It reads the `data-target` attribute that's already on the toggler (`#dvNavCollapse`), finds that element, and toggles the `show` class on click. The Bootstrap 4 SCSS already defines `.collapse.show { display: block; }`, so the visual behaviour is unchanged.

- [ ] **Step 3: Build**

Run: `bundle exec jekyll build --quiet 2>&1 | tail -5`

Expected: clean build.

- [ ] **Step 4: Confirm the CDN scripts are gone from the rendered HTML**

Use Grep with pattern `jquery-3.5.1` path `_site` glob `**/*.html` → expected: no matches anywhere.
Use Grep with pattern `bootstrap.bundle.min.js` path `_site` glob `**/*.html` → expected: no matches.

- [ ] **Step 5: Confirm the new vanilla JS handler is present**

Grep `_site/index.html` for `navbar-toggler` — expected: matches in both the `<button class="navbar-toggler">` markup AND the JS that queries for it.

Also grep for `classList.toggle('show')` → expected: match in the inline script.

- [ ] **Step 6: Commit**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git add _includes/head.html _includes/head/custom.html
git commit -m "perf(head): drop jQuery + Bootstrap JS, vanilla navbar toggle (-86KB)"
```

---

## Task 5: Chrome DevTools MCP verification

**Files:**
- None (verification only)

This task verifies: (a) the fixed tag links actually scroll to the correct sections, (b) the vanilla navbar toggle works on narrow viewports, (c) the site still renders correctly without Bootstrap JS.

- [ ] **Step 1: Start the local Jekyll server in the background**

Run `bundle exec jekyll serve --quiet --port 4000` with `run_in_background: true`.

Verify with `sleep 3 && curl -sI http://localhost:4000 | head -1` → expect `HTTP/1.1 200 OK`.

- [ ] **Step 2: Verify fixed tag link on an article**

Use `navigate_page` to `http://localhost:4000/2026/03/27/what-is-codex-cli/`.

Use `evaluate_script`:
```js
() => {
  const firstTag = document.querySelector('.post-tags .badge');
  return firstTag ? { text: firstTag.textContent.trim(), href: firstTag.getAttribute('href') } : null;
}
```

Expected: `href` starts with `/tags/#tag-` (note the `tag-` prefix).

- [ ] **Step 3: Click the first tag and verify scroll position**

Use `evaluate_script`:
```js
() => {
  const firstTag = document.querySelector('.post-tags .badge');
  const href = firstTag?.getAttribute('href');
  return href;
}
```

Then `navigate_page` to `http://localhost:4000` + the href from the previous step.

Then `evaluate_script`:
```js
() => {
  const hash = location.hash;
  const target = hash ? document.querySelector(hash) : null;
  return {
    hash,
    targetFound: !!target,
    targetText: target?.textContent?.slice(0, 100),
  };
}
```

Expected: `hash` like `#tag-codex-cli`, `targetFound: true`, `targetText` starts with the tag name.

- [ ] **Step 4: Test vanilla navbar toggle on mobile viewport**

Use `emulate` with `viewport: "375x812x2,mobile"`.

Navigate back to the home page: `navigate_page` to `http://localhost:4000/`.

Use `evaluate_script` to check the initial state:
```js
() => {
  const toggler = document.querySelector('.navbar-toggler');
  const collapse = document.querySelector('#dvNavCollapse');
  return {
    togglerVisible: toggler ? getComputedStyle(toggler).display !== 'none' : false,
    collapseHasShow: collapse ? collapse.classList.contains('show') : null,
    collapseVisible: collapse ? getComputedStyle(collapse).display !== 'none' : null,
  };
}
```

Expected: `togglerVisible: true`, `collapseHasShow: false`, `collapseVisible: false` (the collapse is hidden by default).

Now click the toggler by executing its click handler directly (since the MCP click interaction may need an element UID):
```js
() => {
  document.querySelector('.navbar-toggler')?.click();
  const collapse = document.querySelector('#dvNavCollapse');
  return {
    collapseHasShow: collapse?.classList.contains('show'),
    collapseVisible: getComputedStyle(collapse).display !== 'none',
    ariaExpanded: document.querySelector('.navbar-toggler')?.getAttribute('aria-expanded'),
  };
}
```

Expected: `collapseHasShow: true`, `collapseVisible: true`, `ariaExpanded: "true"`.

Click again to close:
```js
() => {
  document.querySelector('.navbar-toggler')?.click();
  const collapse = document.querySelector('#dvNavCollapse');
  return {
    collapseHasShow: collapse?.classList.contains('show'),
    ariaExpanded: document.querySelector('.navbar-toggler')?.getAttribute('aria-expanded'),
  };
}
```

Expected: `collapseHasShow: false`, `ariaExpanded: "false"`.

- [ ] **Step 5: Verify jQuery / Bootstrap JS are NOT loaded**

Use `evaluate_script`:
```js
() => ({
  jquery: typeof window.jQuery,
  $: typeof window.$,
  bootstrap: typeof window.bootstrap,
})
```

Expected: all three are `"undefined"` (no jQuery, no `$`, no Bootstrap global). This proves the vanilla implementation is the only thing handling the navbar.

- [ ] **Step 6: Verify paginated-page description on /page2/**

Reset emulation to desktop: `emulate` with `viewport: "1440x900x1"`.

Navigate: `http://localhost:4000/page2/`.

```js
() => document.querySelector('meta[name="description"]')?.getAttribute('content')
```

Expected: a string ending in `— page 2 of 19` (or similar — match the total number of pagination pages).

Navigate home: `http://localhost:4000/`. Same script. Expected: does NOT contain `— page`.

- [ ] **Step 7: Kill the Jekyll server**

Use `TaskStop` with the background shell ID from Step 1.

- [ ] **Step 8: No commit — verification only**

---

## Task 6: Rebase and push

**Files:**
- None

- [ ] **Step 1: Review local commits**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git log --oneline origin/main..HEAD
```

Expected: 5 commits (Tasks 1, 2, 2a, 3, 4). Tasks 5 and 6 make no commits.

- [ ] **Step 2: Fetch + rebase**

```bash
git fetch origin main
git log --oneline HEAD..origin/main
```

If there are upstream sync-bot commits, rebase:
```bash
git rebase origin/main
```

Expected: clean rebase. None of the files in this PR overlap with `_posts/**` that the sync bot touches.

- [ ] **Step 3: Push**

```bash
git push origin main
```

---

## Self-Review

**Spec coverage:**
| Requirement | Task |
|---|---|
| Fix tag links on articles | Task 1 |
| Fix tag links on home/listings | Task 2 |
| Differentiate paginated descriptions | Task 2a |
| Compress CSS output | Task 3 |
| Drop jQuery + Bootstrap JS bundle | Task 4 |
| Browser-verify tag link, navbar toggle, no jQuery | Task 5 |
| Pushed | Task 6 |

**Placeholder scan:** None. All code shown.

**Type / name consistency:**
- `.navbar-toggler`, `#dvNavCollapse`, `data-target` — used consistently in existing markup (Task 4 reads them, doesn't create new names)
- `paginator.page`, `paginator.total_pages` — standard jekyll-paginate variables (Task 2a)

**Scope check:** Small fixes plus two perf optimisations. No structural changes. No content changes. No new plugins. No new SCSS.

**Risks:**
- Task 3 (SCSS compression): the main risk is a bug in jekyll-sass-converter's minifier that strips or malforms a rule. Mitigation: Task 3 Step 4 greps for three known-critical classes to confirm they survived.
- Task 4 (vanilla navbar): risk is that the Bootstrap SCSS `.collapse.show { display: block; }` rule is actually not present (if, for some reason, the main-site SCSS import doesn't include it). Mitigation: Task 5 Step 4 visually verifies the collapse toggle works.
