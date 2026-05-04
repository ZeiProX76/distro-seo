---
name: distro-seo
description: End-of-audit SEO + Core Web Vitals fixer. Use when given a list of failing audit issues (LCP, INP, CLS, render-blocking, unused JS, image weight, content-to-code ratio, robots.txt, sitemap, meta tags, structured data, AI crawler access, llms.txt, mobile, hreflang) and asked to "solve", "fix", "remediate" or "improve" them. Triggered by phrases like "fix these SEO issues", "solve audit findings", "improve page speed", "fix Core Web Vitals", "remediate Lighthouse failures", "improve SEO score", "fix technical SEO", "boost AI search visibility", "GEO optimization", "fix render-blocking resources".
license: MIT
metadata:
  author: distributionmarket
  version: "1.0.0"
  homepage: https://thedistributionmarket.com
---

# Distro SEO

End-of-audit fixer. You did the audit. This skill turns the issue list into committed fixes.

## When to use

Invoke this skill when the user has an existing audit (Lighthouse, PageSpeed Insights, an SEO crawler report, a third-party tool, or our own DistributionMarket audit) and hands you a list of failing checks. The skill maps each named issue to a diagnostic, a code-first fix, and a verification step. It is framework-aware (Next.js, Astro, SvelteKit, Remix, Vite, plain HTML).

If no audit is in hand, run a baseline first: PageSpeed Insights for the URL, view-source for raw HTML, robots.txt + sitemap.xml + llms.txt fetch. Then come back here.

## How to invoke

The user pastes issues (numbered, comma-separated, or copied from a tool). For each one:

1. **Match it** to an entry in the catalogue below by issue name or symptom.
2. **Diagnose** before fixing. Run the diagnostic snippet to confirm the failure mode.
3. **Apply the fix** for the user's framework. Edit files directly, do not just describe.
4. **Verify** with the post-fix check listed in the entry.
5. **Move on** to the next issue. Do not stop to refactor, do not bundle unrelated cleanup.

Cap each fix to the smallest change that resolves the failing check. Surgical, not aspirational.

## Targets (the bar)

| Metric | Good | Needs work | Poor | Audit threshold |
|---|---|---|---|---|
| LCP | <= 2.5 s | 2.5 to 4 s | > 4 s | mobile p75 |
| INP | <= 200 ms | 200 to 500 ms | > 500 ms | mobile p75 |
| CLS | <= 0.1 | 0.1 to 0.25 | > 0.25 | mobile p75 |
| FCP | <= 1.8 s | 1.8 to 3 s | > 3 s | mobile p75 |
| TTFB | <= 800 ms | 800 to 1800 ms | > 1800 ms | server response |
| Lighthouse Performance (mobile) | >= 90 | 50 to 89 | < 50 | aim for 90+ |
| Lighthouse SEO | 100 | 90 to 99 | < 90 | aim for 100 |
| Content-to-code ratio | >= 10% | 5 to 10% | < 5% | text bytes / total HTML bytes |

Google measures CWV at the **p75 of real user data** (CrUX). Lighthouse lab scores diverge from field. Always confirm with PageSpeed Insights field data after a deploy, not just lab runs.

---

# Issue Catalogue

Each entry below is a self-contained fix recipe. Find the entry by issue name (left column of the index), apply, verify.

## Index

**Performance / CWV**
- 1. Render-blocking resources (CSS, JS, fonts)
- 2. Largest Contentful Paint slow (LCP > 2.5 s)
- 3. Interaction to Next Paint slow (INP > 200 ms)
- 4. Cumulative Layout Shift (CLS > 0.1)
- 5. Time to First Byte slow (TTFB > 800 ms)
- 6. Unused JavaScript
- 7. Unused CSS
- 8. Properly size images
- 9. Serve images in next-gen formats (WebP / AVIF)
- 10. Efficient cache policy on static assets
- 11. Reduce unused third-party code
- 12. Avoid enormous network payloads
- 13. Avoid long main-thread tasks
- 14. Eliminate large layout shifts (sources)
- 15. Web fonts blocking text render

**On-page SEO**
- 16. Title tag missing, generic, or duplicate
- 17. Meta description missing or duplicate
- 18. H1 missing, multiple, or duplicate
- 19. Image alt text missing
- 20. Internal links broken or orphan pages
- 21. Low content-to-code ratio
- 22. Thin content (<300 words on indexable page)
- 23. Open Graph + Twitter Card tags missing

**Technical SEO**
- 24. robots.txt missing, broken, or blocking key paths
- 25. XML sitemap missing or stale
- 26. Canonical tag missing or pointing wrong
- 27. HTTPS / mixed content / redirect chains
- 28. Mobile viewport / tap target / readability
- 29. hreflang implementation broken
- 30. Structured data missing or invalid

**GEO (AI search) — our differentiator**
- 31. AI crawlers blocked in robots.txt
- 32. llms.txt missing or weak
- 33. Content not citable (no clear claims, no source attribution)
- 34. JavaScript-only rendering (AI crawlers do not run JS)

---

## 1. Render-blocking resources

**Symptom**: Lighthouse flags "Eliminate render-blocking resources". CSS in `<head>` and synchronous `<script>` tags before `</body>` delay first paint and LCP.

**Diagnose**:
```bash
# Count blocking requests in head
curl -s "https://example.com" | grep -E '<link[^>]+stylesheet|<script[^>]+src' | grep -v 'defer\|async'
```

**Fix**:
```html
<!-- Inline critical above-the-fold CSS, defer the rest -->
<style>/* critical-path CSS, ideally < 14 KB */</style>
<link rel="preload" href="/styles/main.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
<noscript><link rel="stylesheet" href="/styles/main.css"></noscript>

<!-- Defer all non-critical scripts -->
<script src="/js/analytics.js" defer></script>
<script src="/js/widget.js" async></script>
```

**Next.js**: use `next/script` with `strategy="lazyOnload"` for analytics, `strategy="afterInteractive"` for chat widgets. Move global CSS imports out of `_app.tsx` and into route segments where possible.

```tsx
import Script from 'next/script'
<Script src="https://cdn.example/analytics.js" strategy="lazyOnload" />
```

**Astro**: use `<script is:inline async>` or load via `client:idle` directive for islands.

**Verify**: re-run Lighthouse, the "render-blocking resources" audit should drop to 0 ms or be removed. LCP should fall by 200 to 1500 ms depending on baseline.

---

## 2. LCP > 2.5 s

**Diagnose** (paste in DevTools console):
```js
new PerformanceObserver((list) => {
  const last = list.getEntries().at(-1)
  console.log('LCP:', last.startTime.toFixed(0), 'ms', 'element:', last.element)
}).observe({ type: 'largest-contentful-paint', buffered: true })
```

**Root-cause tree**:

| Sub-cause | Check | Fix |
|---|---|---|
| TTFB > 800 ms | curl -w '%{time_starttransfer}' | Add CDN, cache HTML at edge, reduce server work |
| LCP element is an image | identify via console snippet above | Steps below |
| LCP element is text | font is blocking | Step 15 |
| Render-blocking CSS / JS in head | Step 1 | Step 1 |
| LCP added by JS after hydration | DOM in raw HTML missing the LCP node | Move to SSR / SSG / streaming |

**Image LCP fix**:
```html
<!-- Mark the LCP image -->
<link rel="preload" as="image" href="/hero.avif" fetchpriority="high"
      imagesrcset="/hero-640.avif 640w, /hero-1280.avif 1280w" imagesizes="100vw">
<img src="/hero.avif" alt="..." width="1280" height="720" fetchpriority="high">
```
Never use `loading="lazy"` on the LCP image.

**Next.js**: `<Image priority fetchPriority="high" />` on the hero. Set `sizes` accurately. If the hero is a remote image, add the domain to `next.config.js` `images.remotePatterns`.

**Verify**: LCP < 2.5 s on PSI mobile. Element identified by the observer should match the intended hero.

---

## 3. INP > 200 ms

**Diagnose**:
```js
new PerformanceObserver((list) => {
  for (const e of list.getEntries()) {
    if (e.duration > 200) console.warn('Slow interaction', {
      type: e.name, duration: e.duration, target: e.target,
      processing: e.processingEnd - e.processingStart
    })
  }
}).observe({ type: 'event', buffered: true, durationThreshold: 16 })
```

**Common causes + fixes**:

1. **Long task in the click handler**:
```js
// before: synchronous heavy work
button.addEventListener('click', () => {
  const result = expensiveCompute(items)
  render(result)
})

// after: yield, schedule, give immediate feedback
button.addEventListener('click', async () => {
  button.classList.add('is-loading')
  await new Promise(r => requestAnimationFrame(r))
  const result = expensiveCompute(items)
  render(result)
})
```

2. **React re-render storm**: wrap heavy children in `React.memo`, move state down, use `useDeferredValue` for filter inputs.

3. **Third-party widgets**: defer chat / analytics until idle or first interaction.
```js
const load = () => import('https://widget.example/embed.js')
window.addEventListener('scroll', load, { once: true, passive: true })
window.addEventListener('click', load, { once: true })
```

4. **Large DOM** (> 1500 nodes): paginate, virtualize lists (react-window, vue-virtual-scroller).

**Verify**: trigger the worst interaction with throttled CPU (DevTools 4x slowdown). INP entry should be < 200 ms.

---

## 4. CLS > 0.1

**Diagnose**:
```js
let cls = 0
new PerformanceObserver((list) => {
  for (const e of list.getEntries()) {
    if (!e.hadRecentInput) {
      cls += e.value
      console.log('shift', e.value, e.sources?.map(s => s.node))
    }
  }
}).observe({ type: 'layout-shift', buffered: true })
```

**Causes + fixes**:

| Cause | Fix |
|---|---|
| Images without dimensions | Add `width` + `height` (or `aspect-ratio` CSS) on every `<img>` and `<video>` |
| Web font swap | `font-display: optional`, or match fallback metrics with `size-adjust` |
| Ads / embeds inserted late | Reserve space: `min-height` on the slot + `contain-intrinsic-size` for lazy content |
| Cookie banner pushing content | Render as overlay, not inline |
| Late-loading hero | Skeleton placeholder of identical dimensions |

```css
img, video, iframe { aspect-ratio: attr(width) / attr(height); height: auto; }

@font-face {
  font-family: 'Brand';
  src: url('/fonts/brand.woff2') format('woff2');
  font-display: optional;
  size-adjust: 97.5%;
  ascent-override: 90%;
}
```

**Verify**: CLS < 0.1 on PSI field data. Lab CLS in Lighthouse should be < 0.05.

---

## 5. TTFB > 800 ms

**Diagnose**:
```bash
for i in 1 2 3; do
  curl -o /dev/null -s -w "TTFB: %{time_starttransfer}s  Total: %{time_total}s\n" "https://example.com"
done
```

**Fixes by stack**:
- **Static / SSG**: deploy to a CDN edge (Vercel, Cloudflare Pages, Netlify). TTFB should drop to < 200 ms globally.
- **SSR (Next.js, Nuxt, SvelteKit)**: move to Fluid Compute / Edge runtime when latency-bound, cache responses with `Cache-Control: s-maxage=60, stale-while-revalidate=300`, reduce DB round-trips per render.
- **WordPress / Shopify / monolith**: enable full-page cache, upgrade hosting tier, add Cloudflare in front, reduce plugin count.
- **Database-bound API**: index hot queries, add Redis between API and DB, batch N+1.

**Verify**: PSI field TTFB < 800 ms. Fly probe from 3 regions, confirm consistency.

---

## 6. Unused JavaScript

**Diagnose**: Chrome DevTools → Coverage tab (Cmd+Shift+P → "Coverage") → record page load. Sort by Unused Bytes desc.

**Fixes**:
- **Code-split routes**: dynamic `import()` for non-critical modules.
```js
const Editor = lazy(() => import('./Editor'))
```
- **Tree-shake**: import only what you use. `import { debounce } from 'lodash-es'` not `import _ from 'lodash'`.
- **Replace heavy deps**: moment.js (300 KB) → date-fns or Temporal polyfill. lodash → native methods. axios → fetch.
- **Remove polyfills** for browsers you do not support. Set `browserslist: ["> 0.5%, last 2 versions, not dead"]`.
- **Defer SDKs**: Stripe.js, Intercom, GA — load on interaction.

**Verify**: re-record Coverage, target < 30% unused per file. Lighthouse "Reduce unused JavaScript" should fall under 50 KB.

---

## 7. Unused CSS

**Diagnose**: same Coverage tab, filter by CSS.

**Fixes**:
- **Tailwind / utility CSS**: ensure `content` glob in `tailwind.config.js` covers every template path. Purge in production.
- **Component CSS**: split global stylesheets, scope to components (CSS Modules, styled-components, vanilla-extract).
- **Critical CSS**: inline above-the-fold CSS in `<head>`, async-load the rest (see Step 1).
- **Remove dead frameworks**: if you have Bootstrap loaded for one button, replace the button with custom CSS.

**Tools**: `npx purgecss --css "dist/**/*.css" --content "dist/**/*.html"` for one-shot pruning. `npx critical https://example.com --inline` for critical CSS extraction.

**Verify**: unused CSS bytes < 20 KB.

---

## 8. Properly size images

**Diagnose**:
```bash
# Find images served at sizes far larger than displayed
curl -s "https://example.com" | grep -oE 'src="[^"]+\.(jpg|png|webp|avif)[^"]*"' | sort -u
```

**Fix**:
```html
<img src="/hero-1280.webp" srcset="/hero-640.webp 640w, /hero-1280.webp 1280w, /hero-2560.webp 2560w"
     sizes="(max-width: 768px) 100vw, 1280px" width="1280" height="720" alt="...">
```

**Next.js**: `<Image src=... width=... height=... sizes="(max-width: 768px) 100vw, 50vw" />` does srcset + responsive sizing automatically.

**Astro**: `<Image src={...} widths={[640, 1280, 2560]} sizes="..."  />` from `astro:assets`.

**Verify**: Lighthouse "Properly size images" should be passing. Wasted bytes < 50 KB total.

---

## 9. Serve images in next-gen formats

**Convert in bulk**:
```bash
# AVIF (smallest, best quality, ~50% lighter than JPEG)
for img in src/assets/*.jpg; do
  npx @squoosh/cli --avif '{"cqLevel":33}' "$img" -d src/assets/
done

# WebP fallback
for img in src/assets/*.jpg; do
  cwebp -q 80 "$img" -o "${img%.jpg}.webp"
done
```

**Serve with `<picture>` for fallback**:
```html
<picture>
  <source type="image/avif" srcset="/hero.avif">
  <source type="image/webp" srcset="/hero.webp">
  <img src="/hero.jpg" alt="..." width="1280" height="720">
</picture>
```

Frameworks with built-in image components (Next, Astro, Nuxt, SvelteKit) handle this automatically. Check the config emits both AVIF and WebP.

**Verify**: payload reduction 30 to 70% per image. Lighthouse next-gen format audit passes.

---

## 10. Efficient cache policy on static assets

**Diagnose**:
```bash
curl -I "https://example.com/static/main.js" | grep -i cache-control
```
If it shows `Cache-Control: max-age=0` or no header, fix.

**Fix** (Vercel, vercel.ts):
```ts
import { routes } from '@vercel/config/v1'
export const config = {
  headers: [
    routes.cacheControl('/static/(.*)', { public: true, maxAge: '1 year', immutable: true }),
    routes.cacheControl('/_next/static/(.*)', { public: true, maxAge: '1 year', immutable: true }),
    routes.cacheControl('/(.*\\.(jpg|webp|avif|svg|woff2))', { public: true, maxAge: '1 year', immutable: true }),
  ],
}
```

**Nginx**:
```nginx
location ~* \.(js|css|woff2|webp|avif|jpg|png|svg)$ {
  expires 1y;
  add_header Cache-Control "public, immutable";
}
```

**Cloudflare Pages**: add `_headers` file:
```
/static/*
  Cache-Control: public, max-age=31536000, immutable
```

Hashed filenames (Vite, Next.js, Webpack default) make `immutable` safe.

**Verify**: re-load the asset, headers show `cache-control: public, max-age=31536000, immutable`.

---

## 11. Reduce unused third-party code

**Diagnose**: DevTools → Network → filter by Domain (third party).

**Fixes**:
- **Audit cost**: every third-party script costs LCP + INP. Drop anything below "must have".
- **Self-host fonts** (Google Fonts can add 200 to 500 ms):
```html
<!-- Drop this -->
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Inter">
<!-- Add @font-face from local /fonts -->
```
- **Defer chat**, **analytics**, **A/B testing**:
```html
<script src="/intercom.js" defer></script>
```
- **Facade pattern**: render a static placeholder that boots the real widget on interaction (YouTube embeds, Disqus, etc).

**Verify**: third-party byte budget < 200 KB. Long task count on first load drops.

---

## 12. Avoid enormous network payloads

**Budget**:
- Total transfer < 1.5 MB compressed
- JS < 300 KB compressed
- CSS < 100 KB compressed
- Images < 800 KB total above the fold

**Common offenders + fix**:
- Single hero image > 500 KB → next-gen format + responsive sizing (Steps 8 + 9)
- Single JS bundle > 500 KB → code-split routes, lazy-load editors / charts (Step 6)
- All-in-one icon font (FontAwesome 200 KB+) → SVG sprites or per-icon imports

**Verify**: WebPageTest "Total Byte Weight" under 1.5 MB.

---

## 13. Long main-thread tasks

**Diagnose**:
```js
new PerformanceObserver((list) => {
  for (const e of list.getEntries()) console.warn('long task', e.duration.toFixed(0), 'ms', e)
}).observe({ type: 'longtask', buffered: true })
```

**Fix**:
- Break loops with `await scheduler.yield()` (or `setTimeout(r,0)`) every ~50 ms.
- Move CPU work to a Web Worker.
- Defer hydration of below-the-fold islands (Astro `client:visible`, React Server Components).
- Audit third-party scripts (Step 11).

**Verify**: 0 long tasks > 50 ms in the first 5 s after page load.

---

## 14. Eliminate large layout shift sources

Cross-reference Step 4. The CLS observer logs `e.sources[].node` — those are your culprits. Walk each, apply the matching fix from Step 4's table.

---

## 15. Web fonts blocking text render

**Fix the FOIT**:
```css
@font-face {
  font-family: 'Brand';
  src: url('/fonts/brand.woff2') format('woff2');
  font-display: swap;          /* shows fallback immediately, swaps when font loads */
  /* OR font-display: optional; — no swap, no CLS, fallback wins on slow networks */
}
```

**Preload the critical font** (the one used in the LCP):
```html
<link rel="preload" href="/fonts/brand-regular.woff2" as="font" type="font/woff2" crossorigin>
```

**Subset**: serve only the characters / weights you actually use. `glyphhanger` or `fonttools subset` can cut a font 80%.

**Self-host**: drop the `fonts.googleapis.com` request, ship the woff2 from the same origin.

**Verify**: first-paint text appears within 100 ms. CLS contribution from font swap < 0.05.

---

## 16. Title tag missing, generic, or duplicate

**Audit**:
```bash
# Get titles for every URL in sitemap
curl -s https://example.com/sitemap.xml | grep -oE '<loc>[^<]+' | sed 's|<loc>||' | while read url; do
  title=$(curl -s "$url" | grep -oE '<title>[^<]+' | head -1)
  echo "$url || $title"
done
```

**Rules**:
- 50 to 60 characters
- Unique per page
- Primary keyword near the front
- Brand at the end (homepage exception: brand first)

**Fix templates**:
- Product page: `[Product Name] | [Category] | [Brand]`
- Article: `[Title]: [Subhead] | [Brand]`
- Homepage: `[Brand]: [One-line value prop]`

```html
<title>Premium Blue Widgets | Wholesale Pricing | Acme</title>
```

**Verify**: Lighthouse SEO "Document has a `<title>` element" passes. No duplicates across the site.

---

## 17. Meta description missing or duplicate

**Rules**:
- 150 to 160 characters
- Unique per page
- Includes primary keyword + a CTA or differentiator
- Reflects page content

```html
<meta name="description" content="Wholesale blue widgets at 40% off retail. Free shipping over $50, 30-day returns, 4.9 stars from 12,000 buyers. Shop the full range.">
```

**Verify**: every indexable page has a unique description. Lighthouse SEO check passes.

---

## 18. H1 missing, multiple, or duplicate

**Rules**:
- Exactly one `<h1>` per page
- Matches the search intent and the title tag's core keyword
- Unique per page

```html
<h1>Premium Blue Widgets, Built to Last</h1>
```

**Audit**:
```bash
curl -s https://example.com/page | grep -oE '<h1[^>]*>[^<]+' | wc -l   # should be 1
```

**Verify**: 1 H1 per page, no duplicates, no skipped heading levels (h1 → h2 → h3, never h1 → h3).

---

## 19. Image alt text missing

**Audit**:
```bash
curl -s https://example.com | grep -oE '<img[^>]+>' | grep -v 'alt='
```

**Rules**:
- Decorative images: `alt=""` (empty, intentional)
- Content images: descriptive, includes context (50 to 125 chars)
- Avatars: `alt="Photo of Jane Doe"`
- Logos: `alt="Brand Name logo"`
- No keyword stuffing

```html
<img src="/widget.webp" alt="Blue widget with brushed-aluminum housing, mounted on a wood desk">
```

**Verify**: 0 `<img>` without `alt` attribute. Lighthouse Accessibility "Image elements have `[alt]` attributes" passes.

---

## 20. Internal links broken or orphan pages

**Audit**:
```bash
# Crawl-style broken-link check
npx broken-link-checker https://example.com --recursive
```

**Fix**:
- Repair 404s by either restoring the page or 301-redirecting to the closest match.
- Eliminate orphans: every indexable page should be linked from at least one other indexed page (ideally from the homepage or a hub page within 3 clicks).
- Use descriptive anchor text. "Learn more" → "Read the case study on growth tactics".

**Verify**: 0 internal 404s. Every page reachable from the homepage in <= 3 clicks.

---

## 21. Low content-to-code ratio

**Why it matters**: bloated HTML relative to text reduces semantic density, lowers crawl efficiency, weakens AI citation likelihood. Target >= 10%.

**Diagnose**:
```bash
url="https://example.com"
total=$(curl -s "$url" | wc -c)
text=$(curl -s "$url" | sed 's/<[^>]*>//g' | tr -s ' \n\t' ' ' | wc -c)
echo "ratio: $(echo "scale=3; $text / $total * 100" | bc)%"
```

**Fix**:
- **Surface real data on the page**: turn database stats, FAQ answers, "how it works" copy into visible body text (50 to 100 words per FAQ answer, not 1 sentence).
- **Remove unused CSS / JS** (Steps 6, 7) — markup bloat directly hurts the ratio.
- **Inline only critical CSS** (Step 1) — but balance against the ratio if you go too far.
- **Add an "About the data" or "How it works" section** (150 to 300 words) between hero and main content.
- **Expand FAQ items** with full answers, schema-marked (Step 30).

**Verify**: ratio >= 10%. Recompute with the snippet above.

---

## 22. Thin content (< 300 words on indexable page)

**Audit**:
```bash
curl -s https://example.com/page | sed 's/<[^>]*>//g' | wc -w
```

**Fix**:
- For category / list pages: add a 200-word intro that frames the category.
- For product pages: spec table + use-case section + FAQ.
- For tag pages with no content: noindex them, do not try to bulk up with filler.
- For thin blog posts: merge into a stronger pillar post, 301-redirect the old URL.

**Verify**: every indexable page > 300 words of unique substantive copy.

---

## 23. Open Graph + Twitter Card tags missing

**Fix**:
```html
<meta property="og:title" content="Premium Blue Widgets | Acme">
<meta property="og:description" content="Wholesale pricing, free shipping over $50.">
<meta property="og:image" content="https://example.com/og/widgets.png">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:url" content="https://example.com/widgets">
<meta property="og:type" content="website">

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="Premium Blue Widgets | Acme">
<meta name="twitter:description" content="Wholesale pricing, free shipping over $50.">
<meta name="twitter:image" content="https://example.com/og/widgets.png">
```

**Next.js**: use the `metadata` export in `app/page.tsx`:
```ts
export const metadata = {
  openGraph: { title: '...', description: '...', images: ['/og/widgets.png'] },
  twitter: { card: 'summary_large_image', title: '...', images: ['/og/widgets.png'] },
}
```

**Verify**: paste URL into https://opengraph.dev/, image renders 1200x630.

---

## 24. robots.txt missing, broken, or blocking key paths

**Fetch + lint**:
```bash
curl -s https://example.com/robots.txt
```

**Minimum healthy template**:
```
User-agent: *
Allow: /
Disallow: /admin/
Disallow: /api/internal/

# Allow CSS, JS, images so Google can render the page
Allow: /*.css$
Allow: /*.js$

Sitemap: https://example.com/sitemap.xml
```

**Common breakage**:
- `Disallow: /` (blocks the entire site, fatal)
- Blocking `/static/`, `/_next/`, `/assets/` (kills rendering for Googlebot)
- Missing `Sitemap:` directive
- Wildcards too aggressive

**Verify**: Google Search Console robots.txt tester reports no blocked critical paths. `curl -A "Googlebot" https://example.com` returns 200.

---

## 25. XML sitemap missing or stale

**Audit**:
```bash
curl -s https://example.com/sitemap.xml | head -40
curl -s https://example.com/sitemap.xml | grep -c '<url>'
```

**Rules**:
- One URL per indexable canonical page
- Accurate `<lastmod>` dates
- 50,000 URL / 50 MB cap per file (use a sitemap index for larger sites)
- HTTPS-only URLs

**Next.js (App Router)**:
```ts
// app/sitemap.ts
import type { MetadataRoute } from 'next'
export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const posts = await getPosts()
  return [
    { url: 'https://example.com', lastModified: new Date(), priority: 1 },
    ...posts.map(p => ({ url: `https://example.com/blog/${p.slug}`, lastModified: p.updatedAt })),
  ]
}
```

**Submit**: Google Search Console → Sitemaps → submit `https://example.com/sitemap.xml`. Bing Webmaster Tools too.

**Verify**: sitemap returns 200, parses, URL count matches actual indexable pages.

---

## 26. Canonical tag missing or pointing wrong

**Rules**:
- Every indexable page has a self-referencing canonical: `<link rel="canonical" href="https://example.com/this-page">`
- Absolute URL, HTTPS, no query params (unless the param is part of the canonical version)
- Pagination: each page canonical to itself, not to page 1
- Faceted nav: canonical to the un-faceted version

**Audit**:
```bash
curl -s https://example.com/page | grep -i 'rel="canonical"'
```

**Fix template**:
```html
<link rel="canonical" href="https://example.com/products/blue-widget">
```

**Next.js**:
```ts
export const metadata = { alternates: { canonical: 'https://example.com/products/blue-widget' } }
```

**Verify**: every page has exactly one canonical tag, pointing to the HTTPS, lowercase, trailing-slash-normalized version of itself.

---

## 27. HTTPS / mixed content / redirect chains

**Audit**:
```bash
# Redirect chain
curl -sILo /dev/null -w "%{redirect_url} -> %{http_code}\n" http://example.com

# Mixed content scan
curl -s https://example.com | grep -E 'http://[^"]+'
```

**Fix**:
- Force HTTPS at the edge, single 301 from http to https (no chain).
- Force one canonical host: pick `www` or apex, redirect the other once.
- Replace any in-HTML `http://` reference with `https://` or protocol-relative.
- Add HSTS:
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

**Verify**: `http://example.com` → 301 → `https://example.com` in one hop. Zero `http://` references in HTML. SSL Labs grade A or A+.

---

## 28. Mobile viewport, tap target, readability

**Fix**:
```html
<meta name="viewport" content="width=device-width, initial-scale=1">
```

**Tap targets**: every interactive element >= 44x44 px with >= 8 px gap.
```css
button, a.button { min-height: 44px; min-width: 44px; padding: 12px 16px; }
```

**Readability**: body text >= 16 px, line-height 1.5, contrast ratio >= 4.5:1 (WCAG AA).

**Verify**: Lighthouse Mobile-Friendly + Accessibility passes. Google Mobile-Friendly Test passes.

---

## 29. hreflang implementation broken

**Fix template** (every language variant has the full set, including a self-reference and x-default):
```html
<link rel="alternate" hreflang="en" href="https://example.com/page">
<link rel="alternate" hreflang="fr" href="https://example.com/fr/page">
<link rel="alternate" hreflang="de" href="https://example.com/de/page">
<link rel="alternate" hreflang="x-default" href="https://example.com/page">
```

**Rules**:
- Reciprocal: if EN points to FR, FR must point back to EN.
- Use ISO 639-1 language codes (en, fr, de) and ISO 3166-1 region codes (en-US, en-GB).
- Include `x-default` for the language picker / fallback.
- Self-referencing entry required.

**Verify**: https://technicalseo.com/tools/hreflang/ reports no errors.

---

## 30. Structured data missing or invalid

**Pick the schema** that matches the page intent:
- Homepage: `Organization` + `WebSite` (with `SearchAction`)
- Article / blog post: `Article` (or `BlogPosting`, `NewsArticle`)
- Product: `Product` + `Offer` + `AggregateRating`
- FAQ: `FAQPage`
- Breadcrumb trail: `BreadcrumbList`
- Local biz: `LocalBusiness`
- How-to: `HowTo`

**Template (Article)**:
```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "How to Pick a Blue Widget",
  "image": ["https://example.com/img/widget.webp"],
  "datePublished": "2026-04-01T08:00:00+00:00",
  "dateModified": "2026-04-15T10:00:00+00:00",
  "author": [{"@type": "Person", "name": "Jane Doe", "url": "https://example.com/authors/jane"}],
  "publisher": {"@type": "Organization", "name": "Acme", "logo": {"@type": "ImageObject", "url": "https://example.com/logo.png"}}
}
</script>
```

**Template (FAQ)**:
```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {"@type": "Question", "name": "Where does the data come from?",
     "acceptedAnswer": {"@type": "Answer", "text": "All data is collected from public sources..."}}
  ]
}
</script>
```

**Verify**: paste into https://validator.schema.org/ and https://search.google.com/test/rich-results — 0 errors, 0 warnings.

---

## 31. AI crawlers blocked in robots.txt

**Why**: blocking AI crawlers removes you from ChatGPT, Perplexity, Claude, Gemini, AI Overviews citations. This is a GEO killshot.

**Audit**:
```bash
curl -s https://example.com/robots.txt | grep -iE 'GPTBot|PerplexityBot|ClaudeBot|Google-Extended|CCBot|Bytespider|Applebot-Extended'
```

**The matrix to allow**:

| Crawler | Powers |
|---|---|
| GPTBot | ChatGPT browse + training |
| OAI-SearchBot | ChatGPT search results |
| ChatGPT-User | ChatGPT user-initiated fetches |
| PerplexityBot | Perplexity citations |
| ClaudeBot | Claude browse |
| Google-Extended | Gemini + AI Overviews training |
| CCBot | Common Crawl (training data for many models) |
| Applebot-Extended | Apple Intelligence |
| Bytespider | TikTok / Doubao |
| Amazonbot | Alexa / Rufus |
| FacebookBot | Meta AI |

**Fix** (allow everything by default):
```
User-agent: *
Allow: /
Disallow: /admin/

Sitemap: https://example.com/sitemap.xml
```

If you must opt out of AI training but stay in search:
```
User-agent: Google-Extended
Disallow: /

User-agent: GPTBot
Disallow: /
```
Note: blocking `Google-Extended` does not block `Googlebot`. Search rankings unaffected, but AI Overview eligibility drops.

**Verify**: curl with each AI user-agent returns 200 on key URLs.
```bash
curl -A "GPTBot/1.0" https://example.com/ -o /dev/null -w "%{http_code}\n"
```

---

## 32. llms.txt missing or weak

**What**: `https://example.com/llms.txt` is a Markdown file at the root that summarizes the site for LLMs. Spec: https://llmstxt.org/. Helps Claude, Perplexity, ChatGPT pick the right pages to cite.

**Template** (`/llms.txt`):
```markdown
# Acme

> One-sentence positioning. What you do, who for, what makes you different.

A short paragraph (60-120 words) describing the product, the audience, the data
you publish, and the unique angle. Use real numbers, real categories, real names.

## Docs

- [Quickstart](https://example.com/docs/quickstart): Get started in 5 minutes.
- [API reference](https://example.com/docs/api): All endpoints + auth.

## Data

- [Channel index](https://example.com/channels): 84 distribution channels with cost, effort, ramp time.
- [App profiles](https://example.com/apps): 60 fast-growing apps with peak revenue.

## Policies

- [Terms](https://example.com/terms)
- [Privacy](https://example.com/privacy)
```

**Optional** `/llms-full.txt` — same structure but with the full content of every linked page concatenated, so an LLM can ingest the whole site in one fetch.

**Verify**: `curl https://example.com/llms.txt` returns the file with `Content-Type: text/markdown` or `text/plain`.

---

## 33. Content not citable

**Why AI engines do not cite you**: claims are vague, no source attribution, no data, no expert byline.

**Fix patterns**:
- **Front-load the claim**: "X is Y because Z" in the first paragraph, not the fifth.
- **Quantify**: "Most users" → "62% of users (n=1,184 surveyed Jan 2026)".
- **Cite primary sources**: link the study, the dataset, the public filing. Inline `<cite>` or footnote.
- **Author byline + bio + credentials**: `<meta name="author">` + visible "Written by Jane Doe, ex-Google PM" + Person schema.
- **Datestamp**: visible "Published Jan 2026, last updated Apr 2026" + Article schema dates.
- **Pull quotes for key claims**: makes them easier for LLMs to extract verbatim with attribution.

**Verify**: paste the page into https://www.perplexity.ai/, ask a question that should be answered by the page, see if it cites you. If not, the page is not citable yet.

---

## 34. JavaScript-only rendering

**Symptom**: `view-source` shows an empty `<div id="root">`. Search and AI crawlers see nothing.

**Audit**:
```bash
# What does a no-JS crawler see?
curl -s https://example.com | sed 's/<script[^>]*>.*<\/script>//g' | sed 's/<[^>]*>//g' | tr -s ' \n' ' ' | wc -w
```
Should be > 300 words. If under 50, the page is JS-only.

**Fix**:
- **Next.js**: convert client components to server components. Use the App Router with default RSC. `getServerSideProps` / `getStaticProps` in Pages Router.
- **Astro / SvelteKit / Remix**: SSR / SSG by default, opt into client islands.
- **Plain SPA (CRA, Vite)**: switch to a framework with SSR (Remix, Next, Nuxt) or pre-render with `prerender.io` / `react-snap`.
- **Single-page demo**: at minimum, render an HTML shell with the headline, hero copy, key claims, and links so crawlers see something.

**Verify**: `curl -s https://example.com | grep -i 'your-headline'` returns the actual hero text.

---

# Framework adapters (quick reference)

## Next.js (App Router, recommended)
- Image: `next/image` with `priority` on LCP, `sizes` on responsive
- Script: `next/script` with `strategy` (`lazyOnload`, `afterInteractive`, `worker`)
- Metadata: `metadata` export per route + `generateMetadata` for dynamic
- Sitemap: `app/sitemap.ts` returning `MetadataRoute.Sitemap`
- Robots: `app/robots.ts` returning `MetadataRoute.Robots`
- Fonts: `next/font/google` (auto-self-hosted, zero CLS)
- Caching: `Cache-Control` headers via `headers()` in `next.config.ts`, ISR via `revalidate`
- Rendering: default RSC = server-side. Avoid `'use client'` unless needed.

## Astro
- Image: `astro:assets` with `<Image>` and `<Picture>`, auto AVIF/WebP
- Script: `client:idle`, `client:visible`, `client:media` for islands
- SSR: enable `output: 'server'` or `'hybrid'` in `astro.config.mjs`
- Sitemap: `@astrojs/sitemap` integration

## SvelteKit
- Image: `@sveltejs/enhanced-img`
- SSR / SSG: per-route via `+page.server.ts` and `prerender = true`
- Headers: `setHeaders` in load functions

## Remix
- Image: roll your own with `<picture>` or a CDN like Cloudinary
- Headers: `headers` export per route
- Cache: `Cache-Control` headers with stale-while-revalidate

## Plain HTML
- Use `<picture>` for next-gen formats
- Inline critical CSS in `<style>`
- Defer all non-critical JS
- Hand-write robots.txt + sitemap.xml + llms.txt

---

# Verification loop

After fixing each batch:

1. **Re-run PageSpeed Insights** on the same URL: https://pagespeed.web.dev/analysis?url=...
2. **Compare field data** (CrUX) — note that field data takes 28 days to fully reflect changes. Lab data updates immediately.
3. **Lighthouse CI** in your repo to prevent regression:
```yaml
# .github/workflows/lighthouse.yml
- uses: treosh/lighthouse-ci-action@v12
  with:
    urls: |
      https://example.com
      https://example.com/key-page
    budgetPath: ./budget.json
```
4. **Submit a re-crawl** in Google Search Console → URL Inspection → Request Indexing.
5. **Re-fetch with AI user agents** (Step 31) to confirm you are not blocked.
6. **Re-test rich results** at https://search.google.com/test/rich-results.
7. **Track in PostHog / GA4**: pageviews from organic search, AI referrals (look for `chat.openai.com`, `perplexity.ai`, `bing.com/chat`, `gemini.google.com` in referrers).

---

# Anti-patterns (do not do)

- Refactor the entire site to fix one image. Surgical fixes only.
- Keyword-stuff titles or headings. Algorithms (and users) hate it.
- Add structured data that does not match what is on the page. Google will mark it spam.
- Use `noindex` to "fix" thin content. Either improve the content or 410 the page.
- Block AI crawlers because of "training data" fear. You lose citations and traffic.
- Blindly inline all CSS. > 14 KB of inline CSS hurts LCP more than it helps.
- Set `loading="lazy"` on the LCP image. It pushes LCP later.
- Use `font-display: block`. It causes invisible text and hurts LCP.
- Run Lighthouse once and declare victory. Field data (CrUX) is what Google uses.

---

# Output format

When the user gives you a list of issues, respond per-issue in this shape:

```
## Issue: [name]
**Diagnosed cause**: [one line]
**Files changed**: [list]
**Fix applied**:
[code diff or summary]
**Verification**: [how to confirm]
**Expected impact**: [metric movement, ballpark]
```

End with a one-line summary: "Fixed N of M issues. Skipped: [list with reason]."

If a fix needs human judgment (e.g., "rewrite this thin product page" or "decide if you want to opt out of AI training"), surface the question and stop on that issue rather than guessing.
