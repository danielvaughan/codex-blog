# SEO PR 3 — Internal Linking & Image Hints Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Densify the internal link graph between articles via a tag-overlap "Related posts" block on every post, and add cheap performance/SEO wins for images (LCP hints on the sketchnote hero, `loading="lazy"` on other images, `decoding="async"` on all). All pure Liquid, no custom plugins, no article text changes.

**Architecture:** Two independent deliverables. (1) A new `_includes/related-posts.html` partial computes the top N related posts by tag-overlap score using pure Liquid, wired into `_layouts/post.html` after the article body. (2) `_layouts/post.html` gets a Liquid string-replace pass on `content` that adds `loading="eager" fetchpriority="high" decoding="async"` to sketchnote `<img>` tags (the LCP hero) and `loading="lazy" decoding="async"` to any other `<img>` tags.

**"Cited by" backlinks are NOT in this PR.** Pre-flight scan of `_posts/**/*.md` found **zero** cross-links between posts (no `](/YYYY/...)` patterns, no `{% post_url %}` usage), so a Cited-by block would be empty on every page. Skip until posts start referencing each other. Documented in "Out of scope" below.

**Tech Stack:** Jekyll 3 Liquid. No plugins. No new dependencies. Build must succeed on safe-mode GitHub Pages legacy builder.

**Testing note:** Same as PRs 1+2 — static site with no test framework; verification = build + grep + local serve + browser inspection.

---

## Out of scope (documented)

- **Cited by backlinks** — skipped because current posts don't cross-link. Revisit if the blog adopts cross-references.
- **Explicit `width`/`height` attributes on images** — requires reading image files at build time, which needs a Jekyll plugin not allowed on GitHub Pages legacy builder. CSS `aspect-ratio` would require knowing a single dominant ratio, but sketchnotes split between 16:9 and ~3:2. Deferred.
- **AVIF / WebP image formats** — requires a build-time image pipeline not allowed on legacy builder. Deferred.

---

## File Structure

| File | Action | Responsibility |
|---|---|---|
| `_includes/related-posts.html` | Create | Pure-Liquid tag-overlap scoring; outputs an `<aside class="related-posts">` with the top 5 related posts |
| `_sass/codex-blog.scss` | Modify | Append `// ---------- Related posts ----------` section styling the aside |
| `_layouts/post.html` | Modify | Include the related-posts partial after the body; add Liquid string-replace for image loading attributes |

Three files total. Each has one clear responsibility.

---

## Task 1: Related posts include with tag-overlap scoring

**Files:**
- Create: `/Users/danielvaughan/Development/git/codex-blog/_includes/related-posts.html`

**Context:** Pure Liquid, no plugin. Algorithm: for each post in `site.posts`, count how many tags it shares with the current page; skip if 0 or if it's the same page; build an entry string of form `{padded_score}|{url}|{title}`; push to a list; sort the list lexically in reverse (highest score first); take the first 5; split each entry back into score/url/title for rendering.

The padded-score trick is needed because Liquid's `sort` filter on arrays of strings is lexical — `"10"` sorts before `"2"` unless both are zero-padded to the same width. Five digits of padding covers any realistic score (≤99999 shared tags).

Liquid doesn't have a convenient way to get a post object back from a URL, so we embed the title directly in the entry string to avoid a second lookup pass.

**Step 1: Create the file**

Use Write to create `/Users/danielvaughan/Development/git/codex-blog/_includes/related-posts.html` with this EXACT content:

```liquid
{%- comment -%}
  Related Posts — pure-Liquid tag-overlap scoring.

  For the current `page`, iterates every post in site.posts, counts shared
  tags, and emits an <aside> listing the top 5 most related posts. Each
  entry in the intermediate list is a string of the form
      "{score_padded}|{url}|{title}"
  so that Liquid's lexical sort produces a useful ordering when reversed.

  If the current page has no tags, or there are no related posts with any
  overlap, this include emits an empty string (no aside).

  Performance note: O(posts × tags) per page, ~187 posts × ~7 avg tags ~=
  1300 checks per page, times ~187 post pages = ~243k checks per build.
  Well under a second on Jekyll 3.
{%- endcomment -%}

{%- if page.tags and page.tags.size > 0 -%}
  {%- assign related_entries = "" | split: "" -%}
  {%- for other in site.posts -%}
    {%- if other.url == page.url -%}{%- continue -%}{%- endif -%}
    {%- assign score = 0 -%}
    {%- for tag in other.tags -%}
      {%- if page.tags contains tag -%}
        {%- assign score = score | plus: 1 -%}
      {%- endif -%}
    {%- endfor -%}
    {%- if score > 0 -%}
      {%- assign score_str = score | prepend: "00000" -%}
      {%- assign score_padded = score_str | slice: -5, 5 -%}
      {%- assign entry = score_padded | append: "|" | append: other.url | append: "|" | append: other.title -%}
      {%- assign related_entries = related_entries | push: entry -%}
    {%- endif -%}
  {%- endfor -%}

  {%- if related_entries.size > 0 -%}
    {%- assign sorted = related_entries | sort | reverse -%}
    {%- assign top = sorted | slice: 0, 5 -%}
<aside class="related-posts" aria-label="Related articles">
  <h2 class="related-posts__title">Related articles</h2>
  <ul class="related-posts__list">
    {%- for entry in top -%}
      {%- assign parts = entry | split: "|" -%}
      {%- assign rel_url = parts[1] -%}
      {%- assign rel_title = parts[2] -%}
    <li><a href="{{ rel_url | relative_url }}">{{ rel_title | escape }}</a></li>
    {%- endfor -%}
  </ul>
</aside>
  {%- endif -%}
{%- endif -%}
```

**Step 2: Build and verify Liquid parses**

Run: `cd /Users/danielvaughan/Development/git/codex-blog && bundle exec jekyll build --quiet 2>&1 | tail -5`

Expected: clean build, no Liquid errors. (The file exists but isn't yet wired into any layout, so nothing renders.)

**Step 3: Commit**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git add _includes/related-posts.html
git commit -m "feat(includes): add tag-overlap related-posts partial"
```

Only stage that file.

---

## Task 2: Wire the related-posts partial into post.html and add image loading attributes

**Files:**
- Modify: `/Users/danielvaughan/Development/git/codex-blog/_layouts/post.html`

**Step 1: Read the current layout**

Use Read on `_layouts/post.html`. Current structure:
```html
---
layout: default
---
{% include book-widget.html placement="rail" %}
<article class="post">
  <header class="post-header">
    ...
  </header>

  {% include book-widget.html placement="inline" %}

  <nav class="post-toc" aria-label="On this page" hidden>
    ...
  </nav>

  <section class="post-body">
    {{ content }}
  </section>
</article>
```

**Step 2: Replace `{{ content }}` with a pre-processed variant that injects image loading attributes**

The strategy: before emitting the content, use Liquid's `replace` filter to:
1. First, tag sketchnote images (the hero / LCP image in each post) with `loading="eager" fetchpriority="high" decoding="async"`
2. Then, tag any remaining `<img src="` occurrences with `loading="lazy" decoding="async"`

Order matters: after step 1, the sketchnote lines start with `<img loading="eager" ...`, so the step 2 pattern `<img src="` no longer matches them. Only genuinely-unannotated images (if any) get lazy-loaded.

Use Edit on `_layouts/post.html`:

Old string:
```html
  <section class="post-body">
    {{ content }}
  </section>
</article>
```

New string:
```html
  <section class="post-body">
    {%- comment -%}
      Inject image loading hints:
      - Sketchnote images (/sketchnotes/...) are the hero / LCP image: eager +
        fetchpriority="high" so the browser prioritises them.
      - Any other images get loading="lazy" as a sensible default.
      - All images get decoding="async" for a small perf win.
      Order matters — the sketchnote substitution runs first so its <img>
      tags no longer match the second `<img src="` pattern.
    {%- endcomment -%}
    {%- assign post_content = content | replace: '<img src="/sketchnotes/', '<img loading="eager" fetchpriority="high" decoding="async" src="/sketchnotes/' -%}
    {%- assign post_content = post_content | replace: '<img src="', '<img loading="lazy" decoding="async" src="' -%}
    {{ post_content }}
  </section>

  {% include related-posts.html %}
</article>
```

Note: the related-posts include sits AFTER `</section>` but BEFORE `</article>` — so it's part of the post container (semantically related to the post) but outside the `.post-body` text flow (so its styling is independent).

**Step 3: Build**

Run: `bundle exec jekyll build --quiet 2>&1 | tail -5`

Expected: clean build.

**Step 4: Verify related-posts block appears on a sketchnote post**

Use Grep on `_site/2026/03/27/what-is-codex-cli/index.html`:
- Pattern `class="related-posts"` → expect at least 1 match
- Pattern `Related articles` → expect at least 1 match (the h2)
- Pattern `<li><a href="/2026/` → expect at least 5 matches (up to 5 related posts, assuming the post has enough tag overlap with other posts)

Use Grep on the same file for the image attributes:
- Pattern `loading="eager" fetchpriority="high" decoding="async"` → expect at least 1 match (the sketchnote hero)
- Pattern `src="/sketchnotes/articles/2026-03-27-what-is-codex-cli` → expect match (the sketchnote is still there)

**Step 5: Verify the related-posts block does NOT appear on the home page**

Use Grep on `_site/index.html`:
- Pattern `class="related-posts"` → expect NO match (home page doesn't use post layout)

**Step 6: Performance sanity check — build time**

Run:
```bash
time bundle exec jekyll build --quiet 2>&1 | tail -3
```

Record the total time. Expected: under 30 seconds for a full build of ~187 posts. If it's materially slower than before the related-posts change, that's a red flag — flag it as DONE_WITH_CONCERNS and report the time.

If possible, compare against the previous build time for context. You can get a rough baseline by first reverting the `_layouts/post.html` change, building, measuring, then re-applying and building again. (Optional — skip if time is obviously OK.)

**Step 7: Commit**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git add _layouts/post.html
git commit -m "feat(layout): related-posts block and image loading hints on posts"
```

Only stage `_layouts/post.html`.

---

## Task 3: Style the related-posts block

**Files:**
- Modify: `/Users/danielvaughan/Development/git/codex-blog/_sass/codex-blog.scss` (append new section at end)

**Step 1: Append related-posts styles**

Read the file first to find the end. Then use Edit to append this block at the very end of the file (do NOT replace anything):

The file currently ends with either the book widget section or the pagination section depending on previous edits. Use an anchor at the end — read the last ~20 lines first to find a stable old_string.

Use Edit with the old_string being the very last lines of the file (read them first, then use them verbatim as old_string). The new_string is the same lines plus the appended block:

For example, if the file ends with:
```scss
  &__cta {
    ...existing...
  }
}
```

Your new_string adds the related-posts section after it:
```scss
  &__cta {
    ...existing...
  }
}

// ---------- Related posts ----------
// Sits below the post body as an <aside>. Sober typography, single column,
// minimal decoration — the goal is discoverability, not a visual flourish.
.related-posts {
  margin: 3rem 0 1rem;
  padding-top: 2rem;
  border-top: 1px solid $gray-200;
}
.related-posts__title {
  font-family: $headings-font-family;
  font-weight: 700;
  font-size: 0.9rem;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  color: $gray-700;
  margin: 0 0 1rem;
}
.related-posts__list {
  list-style: none;
  padding: 0;
  margin: 0;
  li {
    padding: 0.35rem 0;
    a {
      color: #0759d9;  // same darker-blue used in .post-toc for 4.5:1 contrast
      text-decoration: none;
      &:hover { text-decoration: underline; }
    }
  }
}
```

**Step 2: Build and verify the CSS compiles**

Run: `bundle exec jekyll build --quiet 2>&1 | tail -5`

Expected: clean build.

Use Grep on `_site/assets/css/main.css`:
- Pattern `.related-posts` → expect match
- Pattern `.related-posts__title` → expect match
- Pattern `.related-posts__list` → expect match

**Step 3: Commit**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git add _sass/codex-blog.scss
git commit -m "feat(style): related-posts block styling"
```

---

## Task 4: End-to-end verification in Chrome DevTools MCP

**Files:**
- None (verification only)

- [ ] **Step 1: Start local Jekyll server in background**

Run `bundle exec jekyll serve --quiet --port 4000` with `run_in_background: true`.

Wait briefly with `sleep 3`, verify with `curl -sI http://localhost:4000 | head -1` → expect `HTTP/1.1 200 OK`.

- [ ] **Step 2: Navigate to a sample article**

Use `mcp__plugin_chrome-devtools-mcp_chrome-devtools__navigate_page` to `http://localhost:4000/2026/03/27/what-is-codex-cli/`.

- [ ] **Step 3: Verify the related-posts block renders**

Use `evaluate_script`:

```js
() => {
  const aside = document.querySelector('aside.related-posts');
  if (!aside) return { present: false };
  const title = aside.querySelector('.related-posts__title')?.textContent?.trim();
  const links = Array.from(aside.querySelectorAll('.related-posts__list a')).map(a => ({
    text: a.textContent.trim(),
    href: a.getAttribute('href'),
  }));
  return { present: true, title, count: links.length, links };
}
```

Expected: `present: true`, `title: "Related articles"`, `count` between 1 and 5, `links` pointing to other posts with tag overlap.

- [ ] **Step 4: Verify image loading attributes on the sketchnote hero**

Use `evaluate_script`:

```js
() => {
  const imgs = Array.from(document.querySelectorAll('.post-body img')).map(img => ({
    src: img.getAttribute('src'),
    loading: img.getAttribute('loading'),
    fetchpriority: img.getAttribute('fetchpriority'),
    decoding: img.getAttribute('decoding'),
  }));
  return imgs;
}
```

Expected: the first (sketchnote) image has `loading: "eager"`, `fetchpriority: "high"`, `decoding: "async"`, and a `src` containing `/sketchnotes/articles/`. Any additional images (rare) should have `loading: "lazy"` and `decoding: "async"`.

- [ ] **Step 5: Take a screenshot showing the related-posts block**

Scroll the page so the related-posts block is visible:

```js
document.querySelector('aside.related-posts')?.scrollIntoView({ behavior: 'instant', block: 'center' });
```

Then `take_screenshot` to `/tmp/related-posts.png`. Confirm visually that the block sits below the article body, has a readable title and ~5 links.

- [ ] **Step 6: Navigate to a post without many tags or overlap, verify graceful degradation**

If a low-overlap post is easy to find, spot-check it. If not, skip — the Liquid guards against both the no-tags and no-overlap cases silently emit nothing.

- [ ] **Step 7: Confirm home page is unchanged (no related-posts block)**

Navigate to `http://localhost:4000/`, re-run the Step 3 evaluate_script. Expected: `present: false`.

- [ ] **Step 8: Kill the Jekyll server**

Use `TaskStop` with the background shell ID from Step 1.

- [ ] **Step 9: No commit — verification only**

---

## Task 5: Rebase and push

**Files:**
- None

- [ ] **Step 1: Review local commits**

```bash
git log --oneline origin/main..HEAD
```

Expected: 3 commits (Tasks 1, 2, 3). Task 4 makes no commits.

- [ ] **Step 2: Fetch + rebase**

```bash
git fetch origin main
git log --oneline HEAD..origin/main
```

If upstream has sync-bot commits, rebase:
```bash
git rebase origin/main
```

Expected: clean rebase. No conflicts expected.

- [ ] **Step 3: Push**

```bash
git push origin main
```

Expected: success.

- [ ] **Step 4: (Optional) Smoke test live deploy**

Wait ~1-2 min for GitHub Pages to rebuild. Then fetch a random article and confirm the related-posts aside and image attributes are present. WebFetch will do — ask it to extract the `<aside class="related-posts">` block and the first `<img>` tag's attributes.

---

## Self-Review

**Spec coverage:**
| Requirement | Task |
|---|---|
| Tag-overlap related posts on every article | Tasks 1 + 2 + 3 |
| Eager/fetchpriority hints on sketchnote hero | Task 2 |
| `loading="lazy"` on other images | Task 2 |
| `decoding="async"` on all images | Task 2 |
| No related-posts on non-post pages | Task 2 (only added to `_layouts/post.html`) |
| Styled related-posts block | Task 3 |
| Browser-verified | Task 4 |
| Pushed | Task 5 |

**Skipped by design (documented in "Out of scope"):**
- Cited-by backlinks (no cross-links to backlink)
- Explicit img width/height (requires plugin)
- AVIF/WebP (requires plugin)

**Placeholder scan:** None.

**Type / name consistency:**
- `.related-posts`, `.related-posts__title`, `.related-posts__list` — BEM, used in Task 1 (include), Task 3 (SCSS), and Task 4 (selectors in evaluate_script).
- `post_content` — local Liquid variable in `_layouts/post.html`, scoped to the one include site.

**Scope check:** PR 3 only. No config changes, no new plugins, no touching `_posts/**`, no touching PR 1 or PR 2 files.
