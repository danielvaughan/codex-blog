# SEO PR 1 — AI Overviews & Social Shares Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Upgrade codex.danielvaughan.com's head metadata, structured data, and social-share infrastructure so articles are eligible for Google AI Overviews, Perplexity/ChatGPT/Claude citations, and proper previews on LinkedIn, X, Slack, Discord, and Mastodon — all without touching article body text.

**Architecture:** The site already uses `jekyll-seo-tag` which covers canonical URL, description, and basic OG/Twitter tags. This PR (a) configures jekyll-seo-tag via `_config.yml` to emit `summary_large_image` Twitter cards, `twitter:site` / `twitter:creator`, and a site-wide default `og:image`; (b) adds a new include `_includes/head/structured-data.html` that emits `BreadcrumbList`, `Person` (with `sameAs`), and `Organization` JSON-LD on top of the `BlogPosting` already emitted by jekyll-seo-tag; (c) upgrades the `robots` meta tag to include `max-image-preview:large`; (d) generates a simple branded 1200×630 OG fallback PNG; (e) surfaces a visible "Last updated" line on articles when `last_modified_at` front matter is set (optional per-post), and wires that into `article:modified_time` too.

**Tech Stack:** Jekyll 3 (GitHub Pages legacy builder, safe-mode), Liquid, `jekyll-seo-tag` v2.8.0, ImageMagick (for the one-off OG fallback image — if unavailable, Python PIL), Chrome DevTools MCP for verification.

**Related research:** `docs/superpowers/plans/2026-04-08-book-widget.md` (prior context — unrelated, just shows the conventions for this repo); the audit + best-practices research is in this conversation and should be re-derivable from the current state of the site.

**Testing note:** Jekyll blog with no unit test framework. "Tests" here mean: building the site, grepping the compiled HTML for expected content, validating JSON-LD structure, and visually spot-checking via Chrome DevTools MCP at https://localhost:4000 after each task. Every task ends with explicit verification steps before committing.

**Author social handles (confirmed by reading `_main-site/`):**
- Twitter/X: `@DanielVaughan` (per `_main-site/_config.yml:4`)
- LinkedIn: `danielpvaughan` → `https://www.linkedin.com/in/danielpvaughan`
- GitHub: `danielvaughan` → `https://github.com/danielvaughan`

---

## File Structure

| File | Action | Responsibility |
|---|---|---|
| `_config.yml` | Modify | Add `twitter:`, `social:`, `author.twitter`, `logo`, `image` (default OG image) fields consumed by jekyll-seo-tag |
| `assets/images/og-default.png` | Create | 1200×630 site-wide fallback OG image |
| `_includes/head.html` | Modify | Upgrade `robots` meta, include the new structured-data partial |
| `_includes/head/structured-data.html` | Create | Emits `BreadcrumbList` (posts only), `Person` with `sameAs`, `Organization` JSON-LD |
| `_layouts/post.html` | Modify | Add visible "Last updated" paragraph when `page.last_modified_at` is set |

Each file has a single clear responsibility. The new include is scoped to structured data only — it does not overlap with jekyll-seo-tag's `BlogPosting` emission.

---

## Task 1: Add SEO config to `_config.yml`

**Files:**
- Modify: `_config.yml` — add four new blocks after the existing `author:` block

- [ ] **Step 1: Read current `_config.yml`**

Use the Read tool on `/Users/danielvaughan/Development/git/codex-blog/_config.yml`. Confirm the current structure has `author:` followed by `book:` followed by `defaults:`. You'll be inserting new keys alongside existing top-level keys.

- [ ] **Step 2: Add `logo`, `image`, `twitter`, and `social` top-level keys**

Use Edit to make the following insertion. The old string (existing, to locate the insertion point) is the `author:` block and the blank line after it:

Old string:
```yaml
# Author
author:
  name: "Daniel Vaughan"
  bio: "Building expertise in agentic software engineering with Codex CLI"
  links:
    - label: "GitHub"
      icon: "fab fa-fw fa-github"
      url: "https://github.com/danielvaughan"
```

New string (adds `twitter:` username field into the author block AND adds four new top-level keys immediately after):
```yaml
# Author
author:
  name: "Daniel Vaughan"
  bio: "Building expertise in agentic software engineering with Codex CLI"
  twitter: "DanielVaughan"
  links:
    - label: "GitHub"
      icon: "fab fa-fw fa-github"
      url: "https://github.com/danielvaughan"

# Logo (consumed by jekyll-seo-tag for Organization schema)
logo: "/assets/images/daniel-vaughan-avatar.jpg"

# Default social share image (consumed by jekyll-seo-tag as og:image fallback
# when a post doesn't specify its own image). Must be 1200×630 for best results
# across LinkedIn, X, Facebook, Slack, Discord, Mastodon.
image: "/assets/images/og-default.png"

# Twitter card defaults (consumed by jekyll-seo-tag)
twitter:
  username: "DanielVaughan"
  card: "summary_large_image"

# Other social profiles — consumed by jekyll-seo-tag for Person.sameAs
# in the generated BlogPosting JSON-LD, and referenced by our custom
# structured-data include for additional schema.
social:
  name: "Daniel Vaughan"
  links:
    - "https://www.linkedin.com/in/danielpvaughan"
    - "https://github.com/danielvaughan"
    - "https://twitter.com/DanielVaughan"
```

- [ ] **Step 3: Build and confirm jekyll-seo-tag picks up the new fields**

Run: `cd /Users/danielvaughan/Development/git/codex-blog && bundle exec jekyll build --quiet 2>&1 | tail -5`

Expected: clean build. Only the pre-existing `faraday-retry` warning line.

- [ ] **Step 4: Grep the compiled home page for the upgraded Twitter card**

Run Grep with pattern `summary_large_image` path `_site/index.html` output_mode `content`. Expected: at least one match — proves jekyll-seo-tag read `twitter.card` from config.

Then Grep for `twitter:creator` in the same file. Expected: match for `content="@DanielVaughan"` — proves jekyll-seo-tag read `twitter.username`.

Then Grep for `"sameAs"` in a rendered article (pick any file from `_site/2026/**/*.html`). Expected: match showing LinkedIn + GitHub + Twitter URLs, because `social.links` is picked up by jekyll-seo-tag's BlogPosting JSON-LD.

Note: `og:image` will NOT yet resolve to anything useful because the file at `/assets/images/og-default.png` doesn't exist yet. The config points at it; Task 2 creates the file. The build will still succeed — jekyll-seo-tag just emits the path as-is.

- [ ] **Step 5: Commit**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git add _config.yml
git commit -m "feat(config): add SEO social/OG defaults for jekyll-seo-tag"
```

Only stage `_config.yml`. Do NOT use `git add -A`.

---

## Task 2: Create the site-wide OG fallback image

**Files:**
- Create: `assets/images/og-default.png`

This task creates a 1200×630 PNG that will be used as the Open Graph fallback image for any page that doesn't have its own `image:` front matter. The image should contain the site name and a one-line tagline against a brand-colored background. We generate it programmatically rather than asking the user to hand-design one.

- [ ] **Step 1: Check which image generation tool is available**

Run these in order; stop at the first that returns successfully:

```bash
command -v convert && convert --version | head -1
command -v magick && magick --version | head -1
command -v python3 && python3 -c "from PIL import Image; print('PIL available')"
```

Record which one is available. If NONE are available, report BLOCKED — the controller will need to install one.

- [ ] **Step 2: Create the image using ImageMagick (if available)**

If `convert` or `magick` was found in Step 1, run:

```bash
cd /Users/danielvaughan/Development/git/codex-blog
# Use `magick` if available, else `convert`
MAGICK_CMD=$(command -v magick || command -v convert)
$MAGICK_CMD -size 1200x630 \
  gradient:'#0869ff-#0053d4' \
  -font Helvetica-Bold -pointsize 82 -fill white -gravity north \
  -annotate +0+180 'Codex Blog' \
  -font Helvetica -pointsize 40 -fill '#e6efff' -gravity north \
  -annotate +0+290 'Agentic Software Engineering with Codex CLI' \
  -font Helvetica -pointsize 28 -fill '#c9d9ff' -gravity south \
  -annotate +0+80 'codex.danielvaughan.com' \
  assets/images/og-default.png
```

Expected: creates `assets/images/og-default.png`. Verify with:

```bash
file assets/images/og-default.png
```

Expected output contains `PNG image data, 1200 x 630`.

- [ ] **Step 3: Alternative — create the image using Python PIL (if ImageMagick unavailable)**

ONLY run this if Step 2 was skipped because ImageMagick wasn't available. Create a small Python script inline:

```bash
cd /Users/danielvaughan/Development/git/codex-blog
python3 <<'PY'
from PIL import Image, ImageDraw, ImageFont
import os

W, H = 1200, 630
img = Image.new('RGB', (W, H), '#0869ff')
draw = ImageDraw.Draw(img)

# Gradient — approximate by drawing a dark overlay from top to bottom
for y in range(H):
    t = y / H
    r = int(8 + (0 - 8) * t)
    g = int(105 + (83 - 105) * t)
    b = int(255 + (212 - 255) * t)
    draw.line([(0, y), (W, y)], fill=(r, g, b))

# Try a nice system font, fall back to default
def load_font(size):
    for candidate in [
        '/System/Library/Fonts/Helvetica.ttc',
        '/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf',
        '/System/Library/Fonts/Supplemental/Arial Bold.ttf',
    ]:
        if os.path.exists(candidate):
            return ImageFont.truetype(candidate, size)
    return ImageFont.load_default()

big = load_font(82)
med = load_font(40)
small = load_font(28)

def center(text, y, font, fill):
    bbox = draw.textbbox((0, 0), text, font=font)
    w = bbox[2] - bbox[0]
    draw.text(((W - w) / 2, y), text, fill=fill, font=font)

center('Codex Blog', 180, big, '#ffffff')
center('Agentic Software Engineering with Codex CLI', 300, med, '#e6efff')
center('codex.danielvaughan.com', H - 100, small, '#c9d9ff')

img.save('assets/images/og-default.png', 'PNG', optimize=True)
print('Wrote', 'assets/images/og-default.png', img.size)
PY
```

Verify with `file assets/images/og-default.png` — expected `PNG image data, 1200 x 630`.

- [ ] **Step 4: Verify the image renders and the build picks it up**

Run: `bundle exec jekyll build --quiet 2>&1 | tail -5`

Expected: clean build.

Then confirm the image is in `_site/`:

```bash
ls -la _site/assets/images/og-default.png
```

Expected: file exists, non-zero size.

Then grep the compiled `_site/index.html` for `og-default`. Expected: at least one match of the form `<meta property="og:image" content="/assets/images/og-default.png"` (or similar; jekyll-seo-tag may absolutize the URL).

- [ ] **Step 5: Commit**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git add assets/images/og-default.png
git commit -m "feat(assets): add 1200x630 OG fallback image"
```

Only stage the new image file.

---

## Task 3: Upgrade the `robots` meta tag

**Files:**
- Modify: `_includes/head.html` line 7

The current tag is `<meta name="robots" content="index,follow"/>`. The 2026-recommended tag includes `max-image-preview:large`, `max-snippet:-1`, and `max-video-preview:-1` — these unlock Google Discover, Google AI Overviews image usage, and maximum snippet length. Required in 2026 for AI citation eligibility.

- [ ] **Step 1: Read `_includes/head.html`**

Use Read on `/Users/danielvaughan/Development/git/codex-blog/_includes/head.html`. Confirm line 7 is `<meta name="robots" content="index,follow"/>`.

- [ ] **Step 2: Upgrade the robots meta**

Use Edit:

Old string:
```html
    <meta name="robots" content="index,follow"/>
```

New string:
```html
    <meta name="robots" content="index,follow,max-image-preview:large,max-snippet:-1,max-video-preview:-1"/>
```

- [ ] **Step 3: Build and grep to confirm the new robots tag lands in output**

Run: `bundle exec jekyll build --quiet 2>&1 | tail -5` — expect clean build.

Run Grep with pattern `max-image-preview:large` path `_site/index.html`. Expected: match. Also run Grep with the same pattern on a sample article under `_site/2026/**/*.html` (use head_limit 1 on the glob). Expected: match.

- [ ] **Step 4: Commit**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git add _includes/head.html
git commit -m "feat(head): upgrade robots meta with max-image-preview:large"
```

---

## Task 4: Create structured-data include

**Files:**
- Create: `_includes/head/structured-data.html`

This include emits three additional JSON-LD blocks that jekyll-seo-tag does not provide: `BreadcrumbList` (for article pages only), a standalone `Person` entity with `sameAs` (links the author entity to GitHub/LinkedIn/X for E-E-A-T signal clarity), and `Organization` for the site. These layer on top of the `BlogPosting` that jekyll-seo-tag already emits.

**Schema.org background the implementer needs:**
- `BreadcrumbList` has an `itemListElement` array, each item is a `ListItem` with `position` (1-based), `name`, and `item` (URL)
- For this site the breadcrumb on a post is: Home → the post itself (2 items)
- `Person.sameAs` is an array of URLs pointing to the same entity on other platforms — Google uses this for entity disambiguation and knowledge-graph matching
- `Organization` on a personal blog mirrors the `Person`; we can still emit it with the site name and logo because jekyll-seo-tag uses a dummy Person if the author isn't an explicit Organization

- [ ] **Step 1: Create the include file**

Use Write to create `/Users/danielvaughan/Development/git/codex-blog/_includes/head/structured-data.html` with this exact content:

```liquid
{%- comment -%}
  Supplementary JSON-LD structured data that layers on top of jekyll-seo-tag's
  BlogPosting/WebSite output. Three blocks:

    1. BreadcrumbList — posts only; Home → post
    2. Person        — author entity with sameAs links for E-E-A-T clarity
    3. Organization  — site-level publisher entity

  jekyll-seo-tag already emits BlogPosting and WebSite; this file intentionally
  does NOT duplicate them.

  Read `site.social.links` for sameAs, `site.author` for the author block, and
  `site.logo` for the organization logo.
{%- endcomment -%}

{%- if page.layout == "post" -%}
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "BreadcrumbList",
  "itemListElement": [
    {
      "@type": "ListItem",
      "position": 1,
      "name": "Home",
      "item": "{{ site.url }}{{ site.baseurl }}/"
    },
    {
      "@type": "ListItem",
      "position": 2,
      "name": {{ page.title | jsonify }},
      "item": "{{ site.url }}{{ site.baseurl }}{{ page.url }}"
    }
  ]
}
</script>
{%- endif -%}

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Person",
  "name": {{ site.author.name | jsonify }},
  "url": "{{ site.url }}{{ site.baseurl }}/",
  "image": "{{ site.url }}{{ site.baseurl }}{{ site.logo }}",
  "jobTitle": {{ site.author.bio | jsonify }},
  "sameAs": [
    {%- for link in site.social.links -%}
      {{ link | jsonify }}{%- unless forloop.last -%},{%- endunless -%}
    {%- endfor -%}
  ]
}
</script>

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": {{ site.title | jsonify }},
  "url": "{{ site.url }}{{ site.baseurl }}/",
  "logo": {
    "@type": "ImageObject",
    "url": "{{ site.url }}{{ site.baseurl }}{{ site.logo }}"
  },
  "sameAs": [
    {%- for link in site.social.links -%}
      {{ link | jsonify }}{%- unless forloop.last -%},{%- endunless -%}
    {%- endfor -%}
  ]
}
</script>
```

- [ ] **Step 2: Verify the include builds cleanly (before wiring it into head.html)**

At this point the new file exists but nothing references it. The build must still succeed.

Run: `bundle exec jekyll build --quiet 2>&1 | tail -5`

Expected: clean build, no Liquid errors. (The file isn't yet used so its output won't appear anywhere — that's fine.)

- [ ] **Step 3: Commit**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git add _includes/head/structured-data.html
git commit -m "feat(includes): add BreadcrumbList/Person/Organization JSON-LD include"
```

Only stage the new include file.

---

## Task 5: Wire the structured-data include into `head.html`

**Files:**
- Modify: `_includes/head.html` — add an include directive after the existing `{%- seo -%}` line

- [ ] **Step 1: Read current `_includes/head.html`**

Use Read on `/Users/danielvaughan/Development/git/codex-blog/_includes/head.html`. Confirm line 8 is `{%- seo -%}` (or its current location after Task 3's changes).

- [ ] **Step 2: Add the include call**

Use Edit:

Old string:
```html
    {%- seo -%}
    <link rel="stylesheet" href="{{ '/assets/css/main.css' | relative_url }}">
```

New string:
```html
    {%- seo -%}
    {%- include head/structured-data.html -%}
    <link rel="stylesheet" href="{{ '/assets/css/main.css' | relative_url }}">
```

- [ ] **Step 3: Build and verify all three JSON-LD blocks land in a rendered article**

Run: `bundle exec jekyll build --quiet 2>&1 | tail -5` — expect clean build.

Run Grep on a sample article file (pick any from `_site/2026/**/*.html`) for these patterns, each should match:

- `"@type": "BreadcrumbList"`
- `"@type": "Person"`
- `"@type": "Organization"`
- `"sameAs"` — expect this to match twice (once in Person, once in Organization)

Also confirm that `"@type": "BlogPosting"` still matches (jekyll-seo-tag's original output is not broken).

Then read that full article HTML file and visually inspect the four JSON-LD blocks. Confirm:
- BreadcrumbList has 2 items (Home + post title)
- Person.sameAs contains 3 URLs (LinkedIn, GitHub, Twitter)
- Organization.sameAs contains the same 3 URLs
- Organization.logo.url is the full absolute path to `daniel-vaughan-avatar.jpg`

- [ ] **Step 4: Verify BreadcrumbList does NOT appear on non-post pages**

Run Grep with pattern `"@type": "BreadcrumbList"` path `_site/index.html`. Expected: no match (home page is not a post).

Also Grep on `_site/tags/index.html`. Expected: no match.

This confirms the `{% if page.layout == "post" %}` guard works correctly.

- [ ] **Step 5: Validate JSON-LD syntactically**

Pick one article and extract the three JSON-LD blocks, then parse each as JSON using `python3 -c "import json,sys; json.loads(sys.stdin.read())"`. If any block fails to parse, that's a bug — re-read the include file for Liquid filter errors, missing commas, or unescaped quotes.

Practical command:

```bash
cd /Users/danielvaughan/Development/git/codex-blog
python3 - <<'PY'
import re, json, sys, glob
path = sorted(glob.glob('_site/2026/**/*.html', recursive=True))[0]
print(f'Checking {path}')
html = open(path).read()
blocks = re.findall(r'<script type="application/ld\+json">\s*(.*?)\s*</script>', html, re.DOTALL)
print(f'Found {len(blocks)} JSON-LD blocks')
for i, b in enumerate(blocks, 1):
    try:
        parsed = json.loads(b)
        t = parsed.get('@type', '?')
        print(f'  Block {i}: OK  @type={t}')
    except json.JSONDecodeError as e:
        print(f'  Block {i}: FAIL  {e}')
        print(f'  Content: {b[:200]!r}')
        sys.exit(1)
print('All blocks parse cleanly')
PY
```

Expected output: `All blocks parse cleanly` with 4 blocks (BlogPosting, BreadcrumbList, Person, Organization).

- [ ] **Step 6: Commit**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git add _includes/head.html
git commit -m "feat(head): wire structured-data include into head template"
```

---

## Task 6: Surface "Last updated" line on posts when set

**Files:**
- Modify: `_layouts/post.html` — add a conditional `<time>` element inside `.post-meta` that shows the last-updated date when front matter has `last_modified_at`
- No front matter changes to individual posts — this only activates when a post opts in by adding `last_modified_at:` to its front matter

This task does NOT change any article content. It just makes the layout ready to display an update date when the author adds one. jekyll-seo-tag also picks up `page.last_modified_at` automatically and emits `article:modified_time` and `dateModified` in JSON-LD — so this one front-matter field unlocks both the on-page signal AND the machine-readable signal at the same time. No extra config needed.

- [ ] **Step 1: Read `_layouts/post.html`**

Use Read on `/Users/danielvaughan/Development/git/codex-blog/_layouts/post.html`. Find the `<p class="post-meta">` block — this is where the publish date is shown. You'll add a conditional sibling paragraph for the update date.

- [ ] **Step 2: Add the conditional "Last updated" line**

Use Edit to make this change.

Old string (the existing post-meta block):
```html
    <p class="post-meta">
      <time datetime="{{ page.date | date_to_xmlschema }}">
        {{ page.date | date: "%B %-d, %Y" }}
      </time>
    </p>
```

New string (adds a conditional second `<p>` only when `last_modified_at` is set AND is later than `page.date`):
```html
    <p class="post-meta">
      <time datetime="{{ page.date | date_to_xmlschema }}">
        {{ page.date | date: "%B %-d, %Y" }}
      </time>
    </p>
    {%- if page.last_modified_at and page.last_modified_at != page.date -%}
    <p class="post-meta post-meta-updated">
      Updated <time datetime="{{ page.last_modified_at | date_to_xmlschema }}">{{ page.last_modified_at | date: "%B %-d, %Y" }}</time>
    </p>
    {%- endif -%}
```

- [ ] **Step 3: Build — nothing should change yet on any rendered post**

Run: `bundle exec jekyll build --quiet 2>&1 | tail -5` — expect clean build.

Since no existing post has `last_modified_at` in front matter, the conditional is always false. Grep a sample article for `post-meta-updated` — expect NO matches anywhere.

This is the correct outcome. The template is now ready; opt-in happens post-by-post when the author edits something and adds `last_modified_at: YYYY-MM-DD` to that post's front matter.

- [ ] **Step 4: Smoke test by temporarily adding `last_modified_at` to one post**

Pick one post file under `_posts/` and temporarily add `last_modified_at: 2026-04-07` to its front matter — but DO NOT commit this change. The purpose is verification only.

Run: `bundle exec jekyll build --quiet 2>&1 | tail -5`

Grep the rendered HTML file for that post for `post-meta-updated`. Expected: one match, and near it a visible "Updated April 7, 2026" line.

Also grep for `article:modified_time` in the same file. Expected: match — proves jekyll-seo-tag also picked up the new field automatically and added the OG tag.

Finally grep for `"dateModified"` — expected: match inside the BlogPosting JSON-LD.

- [ ] **Step 5: Revert the smoke-test front matter change**

Use Edit to remove the `last_modified_at:` line you added in Step 4. The file should be byte-for-byte identical to its pre-test state. Verify with:

```bash
git status _posts/
```

Expected: nothing to commit.

- [ ] **Step 6: Commit**

```bash
cd /Users/danielvaughan/Development/git/codex-blog
git add _layouts/post.html
git commit -m "feat(layout): show Updated date when last_modified_at is set"
```

Only stage `_layouts/post.html`.

---

## Task 7: End-to-end verification in Chrome DevTools MCP

**Files:**
- None (verification only)

This task does a full live-browser smoke test of the deployed-looking output via a local `bundle exec jekyll serve`. It verifies all of the above changes land correctly and nothing was broken by them.

- [ ] **Step 1: Start the local Jekyll server in the background**

Run `bundle exec jekyll serve --quiet --port 4000` with `run_in_background: true`.

Wait briefly with `sleep 3`, then verify the server responds:
```bash
curl -sI http://localhost:4000 | head -1
```
Expected: `HTTP/1.1 200 OK`.

- [ ] **Step 2: Navigate to a sample post in Chrome DevTools MCP**

Use `mcp__plugin_chrome-devtools-mcp_chrome-devtools__navigate_page` with URL `http://localhost:4000/2026/03/27/what-is-codex-cli/`.

- [ ] **Step 3: Extract and check all head metadata**

Use `evaluate_script` with:

```js
() => {
  const get = (sel) => document.querySelector(sel)?.getAttribute('content') ?? null;
  const link = (sel) => document.querySelector(sel)?.getAttribute('href') ?? null;
  return {
    robots: get('meta[name="robots"]'),
    ogImage: get('meta[property="og:image"]'),
    ogType: get('meta[property="og:type"]'),
    ogTitle: get('meta[property="og:title"]'),
    ogUrl: get('meta[property="og:url"]'),
    twCard: get('meta[name="twitter:card"]'),
    twSite: get('meta[name="twitter:site"]'),
    twCreator: get('meta[name="twitter:creator"]'),
    twImage: get('meta[name="twitter:image"]'),
    canonical: link('link[rel="canonical"]'),
    description: get('meta[name="description"]'),
    articlePublished: get('meta[property="article:published_time"]'),
    articleAuthor: get('meta[property="article:author"]'),
  };
}
```

Expected:
- `robots`: includes `max-image-preview:large`
- `ogImage`: ends with `/assets/images/og-default.png`
- `ogType`: `article`
- `ogTitle`: the article title
- `twCard`: `summary_large_image`
- `twSite`: `@DanielVaughan`
- `twCreator`: `@DanielVaughan`
- `twImage`: ends with `/assets/images/og-default.png`
- `canonical`: absolute URL to the article
- `description`: non-empty string
- `articlePublished`: an ISO-8601 date

If any of these are missing, report which and which task introduced the gap.

- [ ] **Step 4: Extract all JSON-LD blocks and count them**

Use `evaluate_script`:

```js
() => {
  const scripts = document.querySelectorAll('script[type="application/ld+json"]');
  return Array.from(scripts).map(s => {
    try {
      const d = JSON.parse(s.textContent);
      return d['@type'];
    } catch (e) {
      return 'PARSE_ERROR: ' + e.message;
    }
  });
}
```

Expected: 4-element array `["BlogPosting", "BreadcrumbList", "Person", "Organization"]` (order may vary, and BlogPosting may alternatively be labelled `"BlogPosting"` exactly or may have additional type fields — the important thing is all four types appear).

If any block shows `PARSE_ERROR`, dig into the include, fix it, rebuild, re-verify.

- [ ] **Step 5: Verify OG image actually loads**

Use `evaluate_script`:

```js
async () => {
  const url = document.querySelector('meta[property="og:image"]')?.getAttribute('content');
  if (!url) return { error: 'no og:image' };
  try {
    const res = await fetch(url);
    return { url, status: res.status, type: res.headers.get('content-type') };
  } catch (e) {
    return { url, error: e.message };
  }
}
```

Expected: `status: 200`, `type: image/png`. Proves the OG image asset is actually reachable at the URL jekyll-seo-tag pointed at.

- [ ] **Step 6: Navigate to the home page and re-check**

Use `navigate_page` to `http://localhost:4000/`. Re-run Step 3's script. Expected:
- `ogType`: `website` (not article)
- `ogImage`: same fallback
- `twCard`: `summary_large_image`
- `articlePublished`: null (home is not an article)

Also re-run Step 4. Expected: 3 blocks — `WebSite`, `Person`, `Organization`. No `BreadcrumbList` on home, no `BlogPosting`.

- [ ] **Step 7: Validate structured data via the Schema Markup Validator**

Use `WebFetch` on `https://validator.schema.org/#url=https%3A%2F%2Fcodex.danielvaughan.com%2F2026%2F03%2F27%2Fwhat-is-codex-cli%2F` asking it to summarise any errors or warnings. Note that this validates the LIVE deployed site, not the local build — so the results reflect the current state of `https://codex.danielvaughan.com/`, not your local changes. Skip this step if the changes have not been deployed yet; otherwise include the result in the report.

This is informational only — do not fail the task on warnings alone. Note any errors for a follow-up fix.

- [ ] **Step 8: Kill the Jekyll server**

Use `KillShell` (or `TaskStop`) with the background shell ID from Step 1.

- [ ] **Step 9: No commit — verification only**

---

## Task 8: Rebase and push

**Files:**
- None

- [ ] **Step 1: Review local commits**

Run: `git log --oneline origin/main..HEAD`

Expected: 6 commits, one per Task 1-6. Tasks 7 and 8 make no commits.

- [ ] **Step 2: Fetch origin and check for upstream changes**

Run: `git fetch origin main && git log --oneline HEAD..origin/main`

If upstream has sync-bot commits, rebase:

```bash
git rebase origin/main
```

Expected: clean rebase. The sync bot only touches `_posts/` and sync manifests, so it cannot conflict with any of this PR's files.

- [ ] **Step 3: Push**

Run: `git push origin main`

Expected: success. If rejected because more upstream commits arrived during Step 2, fetch + rebase + push again.

- [ ] **Step 4: Smoke-test the live deploy (optional, 1-2 min wait)**

After pushing, wait for GitHub Pages to rebuild (~1-2 minutes). Then `WebFetch` on `https://codex.danielvaughan.com/2026/03/27/what-is-codex-cli/` asking it to list all `<meta>` tags whose name or property contains `image`, `og`, `twitter`, or `robots`. Confirm the new values landed in production.

Also `WebFetch` on `https://codex.danielvaughan.com/assets/images/og-default.png` to confirm the image is live.

---

## Self-Review

**Spec coverage check (vs PR 1 goals from the audit):**

| Requirement | Task |
|---|---|
| `max-image-preview:large` on every page | Task 3 |
| `twitter:card = summary_large_image` | Task 1 (config-driven via jekyll-seo-tag) |
| `twitter:site` and `twitter:creator` | Task 1 (via `twitter.username` in config) |
| `twitter:image` | Task 1 + Task 2 (fallback from `site.image`) |
| `twitter:description` | Already emitted by jekyll-seo-tag; Task 1 upgrades to `summary_large_image` card which carries description |
| Site-wide fallback `og:image` | Task 1 (config) + Task 2 (asset) |
| `article:modified_time` | Task 6 (front matter is opt-in; jekyll-seo-tag emits the tag automatically when present) |
| `BreadcrumbList` JSON-LD | Tasks 4 + 5 |
| `Person` with `sameAs` | Tasks 4 + 5 |
| `Organization` JSON-LD | Tasks 4 + 5 |
| Visible "Last updated" date on page | Task 6 |
| End-to-end browser validation | Task 7 |

All PR 1 requirements mapped. No gaps.

**Placeholder scan:** No TBDs, no "handle errors appropriately", no vague instructions. Every code step has the full code. Every command has expected output described.

**Type / name consistency:**
- Class / field names consistent across tasks: `site.logo`, `site.image`, `site.twitter.username`, `site.twitter.card`, `site.social.links`, `site.author.twitter`, `page.last_modified_at`.
- CSS class `post-meta-updated` introduced in Task 6 — no other task references it, no conflict.
- The new include path `_includes/head/structured-data.html` is used consistently in Tasks 4 and 5.

**Scope check:** Focused on PR 1 of the audit (AI Overviews + social shares). Does not touch article text, does not add analytics, does not touch robots.txt (that's PR 2), does not add related-posts or backlinks (that's PR 3), does not touch the book widget or pagination.
