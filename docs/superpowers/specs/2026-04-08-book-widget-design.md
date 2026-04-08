# Book Widget — Design

**Date:** 2026-04-08
**Status:** Draft, pending review

## Goal

Give visitors reading any article (or browsing the home page) on `codex.danielvaughan.com` an easy, always-visible path to buy the companion book *Codex CLI: Agentic Engineering from First Principles* on Leanpub. The widget must be unobtrusive enough not to interfere with reading, but persistent enough that the CTA is never more than a glance away.

## Non-goals

- No widget on the tags page (`/tags/`) or individual tag pages — those are drill-down navigation surfaces and should stay clean.
- No A/B testing framework. We ship a single variant; analytics come from Leanpub's own referrer + UTM data.
- No dismiss/close button on the widget. Persistent by design.
- No bottom-of-article CTA in the first iteration (may be added later based on click data).
- No affiliate tracking beyond UTM parameters.

## User-visible behaviour

### Desktop (≥1480px viewport width)

A sticky "book rail" floats in the left viewport margin, outside the Bootstrap `.container` that holds the article. It sits ~90px from the top of the viewport (below the sticky navbar) and stays in place as the reader scrolls. The rail is 150px wide and contains, top to bottom:

1. Book cover image, hotlinked from Leanpub's CDN (`https://d2sofvawe08yqg.cloudfront.net/codex-cli/s_hero`)
2. Book title — "Codex CLI"
3. Subtitle — "Agentic Engineering from First Principles"
4. Price line — "From $19 · pay what you want"
5. CTA button — "Buy on Leanpub →" (white text on `$primary` blue)

Both the cover image and the CTA button link to the Leanpub page. The article's own width is unchanged — the rail lives in the existing left margin, so readers on the widest breakpoint see the same line length they get today.

### Narrow desktop / tablet / mobile (<1480px viewport width)

The floating rail is hidden (there is no free margin to float into). In its place, a horizontal inline card is rendered inside the article content, immediately after the post header (on articles) or immediately after the "About this blog" band (on the home page and paginated listings). The inline card uses the same colours but a horizontal layout: small cover on the left (~80px), title + subtitle + price + CTA on the right.

### Scope of pages

- `_layouts/post.html` — every individual article
- `_layouts/home.html` — home page and all paginated listings (`/`, `/page2/`, …)
- Not `_layouts/page.html` (tags and any standalone pages)

## Architecture

### New and modified files

| File | Change | Purpose |
|---|---|---|
| `_config.yml` | Add `book:` block | Book metadata (title, subtitle, price text, cover URL, Leanpub URL) — data-only, editable without touching markup |
| `_includes/book-widget.html` | New | Reusable widget markup; accepts a `placement` parameter (`"rail"` or `"inline"`) |
| `_sass/codex-blog.scss` | Add `// ---------- Book widget ----------` section | Two scoped selectors (`.book-rail`, `.book-inline`) with breakpoint-gated display |
| `_layouts/post.html` | Include widget twice (rail + inline) | Renders both variants; CSS hides the wrong one |
| `_layouts/home.html` | Include widget twice (rail + inline) | Same as above |

### Data flow

1. `_config.yml` carries the book metadata as site data.
2. Each layout calls `{% include book-widget.html placement="rail" %}` and `{% include book-widget.html placement="inline" %}`.
3. The include reads `site.book.*` and the `include.placement` parameter to build the markup and the UTM-tagged Leanpub URL.
4. At render time, both variants ship in the HTML; CSS media queries decide which is visible.

### Why render both and hide via CSS

- No JavaScript required — works with JS disabled and before hydration.
- No flash of the wrong variant on breakpoint changes (e.g. device rotation).
- The rendering cost is negligible: ~1KB of duplicated markup per page.
- Keeps the Liquid include simple — no viewport detection, no conditional logic in the layouts.

## Configuration

### `_config.yml` addition

```yaml
book:
  title: "Codex CLI"
  subtitle: "Agentic Engineering from First Principles"
  price_text: "From $19 · pay what you want"
  cover_url: "https://d2sofvawe08yqg.cloudfront.net/codex-cli/s_hero"
  leanpub_url: "https://leanpub.com/codex-cli/"
```

### UTM parameters

All Leanpub links carry:

- `utm_source=codex-blog`
- `utm_campaign=book-widget`
- `utm_medium={{ include.placement }}` — either `rail` or `inline`

This lets the author distinguish rail (desktop, always-visible) conversions from inline (mobile/narrow, scroll-past) conversions in Leanpub's referrer analytics.

The full URL built inside the include:
```liquid
{{ site.book.leanpub_url }}?utm_source=codex-blog&utm_medium={{ include.placement }}&utm_campaign=book-widget
```

## CSS strategy

### Desktop rail (`.book-rail`)

```scss
// Custom breakpoint: the rail must fit entirely on screen AND not overlap
// the article container. With a 1140px container, 150px rail, and 20px gap,
// the minimum viewport where the rail's left edge is ≥ 0 is:
//   (100vw - 1140) / 2 - 170 ≥ 0  →  100vw ≥ 1480px
$book-rail-breakpoint: 1480px;

.book-rail {
  display: none;                             // hidden below custom breakpoint
  @media (min-width: #{$book-rail-breakpoint}) {
    display: block;
    position: fixed;
    top: 90px;                               // below the sticky navbar
    left: calc((100vw - 1140px) / 2 - 170px);
    width: 150px;
    z-index: 10;
  }
}
```

The `calc()` math: `(100vw - 1140px) / 2` is the free margin on the left of Bootstrap's 1140px `xl` container. Subtracting 170px (150px rail + 20px gap) puts the rail's right edge 20px left of the container, and its left edge ≥ 0 whenever viewport ≥ 1480px. The media-query guard ensures the rail only appears at viewports wide enough for it to fit entirely on screen.

### Mobile inline card (`.book-inline`)

```scss
.book-inline {
  display: flex;                             // visible by default
  flex-direction: row;
  align-items: center;
  gap: 1rem;
  background: #fff;
  border: 1px solid $gray-200;
  border-radius: $border-radius;
  padding: 1rem;
  margin-bottom: 2rem;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
  @media (min-width: #{$book-rail-breakpoint}) {
    display: none;                           // hidden when the rail takes over
  }
}
```

Because `display: none` also removes elements from the accessibility tree, a screen reader at any viewport width encounters exactly one `<aside>` landmark — never both. This is what lets us safely ship both variants in the same HTML.

### Shared elements

Both variants share:

- White background with subtle shadow
- Cover image with `box-shadow: 0 2px 8px rgba(0, 0, 0, 0.18)`
- Title in Montserrat (inherited from `$headings-font-family`)
- CTA button — `background: $primary; color: #fff;` — same token as the pagination buttons; contrast ~5.17:1, passes WCAG AA
- Hover on the CTA uses `darken($primary, 10%)` for feedback

## Accessibility

- The widget is wrapped in `<aside aria-label="Buy the book: Codex CLI">` — declares itself as a complementary landmark that screen readers can navigate to or skip
- Cover image has descriptive alt text: `alt="Cover of the book Codex CLI by Daniel Vaughan"`
- Cover image is a link with `tabindex="-1"` — clickable/tappable but not in the keyboard tab order, so keyboard users don't encounter the same destination twice in a row
- CTA button is a real `<a>` styled as a button, with `rel="noopener"` on the external link
- All text inside the widget has contrast ≥4.5:1 against its background (verified against `$primary`, `$gray-800`, and white backgrounds during implementation)

## Testing and verification

1. **SCSS build**: `bundle exec jekyll build` succeeds, the new `.book-rail` / `.book-inline` rules appear in `_site/assets/css/main.css`, and no existing selectors are broken.
2. **Visual regression across breakpoints**: use Chrome DevTools MCP to visit the home page and a sample article at four widths:
   - **1600px**: floating rail visible on the left, article unchanged
   - **1479px** (just below breakpoint): floating rail hidden, inline card visible
   - **1024px**: inline card visible at top of content
   - **375px**: inline card visible, still readable, cover and CTA both tappable
3. **Link correctness**: inspect the rendered HTML and confirm both CTAs resolve to
   `https://leanpub.com/codex-cli/?utm_source=codex-blog&utm_medium=rail&utm_campaign=book-widget`
   and
   `https://leanpub.com/codex-cli/?utm_source=codex-blog&utm_medium=inline&utm_campaign=book-widget`
   respectively.
4. **Contrast check**: verify the CTA button still clears WCAG AA (≥4.5:1) using `evaluate_script` to compute the computed contrast ratio.
5. **Scope check**: navigate to `/tags/` and confirm neither variant of the widget appears.
6. **Accessibility spot check**: confirm the `<aside>` landmark is announced and the keyboard tab order visits only the CTA button (not the cover link).

## Open implementation risks

- **Rail only appears on ≥1480px viewports**: this is a significant chunk of laptops (1440×900 MacBook Airs, for example, don't quite qualify). That's a deliberate trade-off in exchange for keeping the article width unchanged — users on narrower screens still see the inline card at the top of the article, so the CTA is never hidden. If click analytics show poor desktop rail exposure, we can revisit by either (a) narrowing the rail to 120px (minimum viewport drops to ~1420px), or (b) tightening the gap to the container.
- **Hotlink stability**: the Leanpub CDN URL (`d2sofvawe08yqg.cloudfront.net/codex-cli/s_hero`) is not a documented API and could change without notice. Mitigation: the URL lives in `_config.yml`, so a future swap (to a committed asset or a new hotlink) is a one-line change. If the URL breaks, we can self-host with a follow-up commit.
- **Sticky navbar height changes**: the `top: 90px` offset assumes the current navbar height. If the navbar is ever restyled taller, the rail may crowd into it. Not a blocker — easy to adjust.

## Future work (explicitly out of scope for this spec)

- Bottom-of-article CTA as a second nudge
- A/B testing different price wordings
- Self-hosting the cover image
- Per-article override to disable the widget on specific posts (e.g. a post about a different book)
- Dismiss button with localStorage memory
