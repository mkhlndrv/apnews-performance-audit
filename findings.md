# Findings

Grouped by area, corrective findings first within each. Each finding is a
distinct, independently observable problem — where several causes feed the same
symptom, they sit under one finding. Rendering numbers come from the PSI mobile
Insights/Diagnostics, networking from the DevTools Network panel, mobile from the
throttled-mobile runs (Slow 4G + 4× CPU), accessibility from a Lighthouse
accessibility audit, build-output findings from inspecting the shipped bundles, and
frames/layers findings from a Performance trace. See [baseline](baseline.md).

## Prioritization (RICE)

Corrective findings are ranked with **RICE** — Reach × Impact × Confidence ÷
Effort — which gives one number so unrelated fixes (an image attribute, an
accessibility label, an ad-stack cut) can be compared directly. It's deliberately
subjective; the point is a consistent, written-down basis for "what first."

Scales, tuned to this audit:

- **Reach (1–10)** — how much of the audience hits it. 10 = every mobile load (the
  primary profile); ~7 = a large subset (first visits, repeat visitors); ~3 = a
  specific group (assistive-tech users, one page).
- **Impact (0.5–3)** — severity per affected user. 3 = massive (blocks the main
  content), 2 = high, 1 = medium, 0.5 = minor.
- **Confidence (0.5–1.0)** — how sure the fix lands, given the evidence and how
  much sits outside first-party control. 1.0 = measured and fully in our hands,
  0.8 = solid, 0.6 = depends on third parties or a business call.
- **Effort (1–5)** — rough person-weeks. 1 = an attribute or two; 3 = a
  build/third-party loading rework; 5 = an org-level program (renegotiating the
  ad stack).

Score = (Reach × Impact × Confidence) ÷ Effort. Higher = do sooner. Good findings
aren't scored — they're observations, not work.

| # | Finding | R | I | C | E | Score | Tier |
|---|---------|:-:|:-:|:--:|:-:|:-----:|:----:|
| 1 | Main article image loads late (LCP) | 10 | 3 | 0.8 | 1 | **24.0** | High |
| 2 | Blank screen before render (FCP) | 10 | 2 | 0.8 | 3 | **5.3** | High |
| 3 | CSS bundle ships ~770 KB, mostly unused | 10 | 2 | 0.7 | 3 | **4.7** | High |
| 4 | Barely loads on a real mobile connection | 10 | 3 | 0.7 | 5 | **4.2** | High |
| 5 | Responsive images set `srcset` but no `sizes` | 9 | 1 | 0.9 | 2 | **4.1** | High |
| 6 | Critical CSS not extracted; full sheet blocks paint | 10 | 2 | 0.6 | 3 | **4.0** | High |
| 7 | Scrolling drops frames (~28 fps) | 8 | 2 | 0.7 | 3 | **3.7** | Med |
| 8 | Tap targets too small / crowded | 7 | 1 | 1.0 | 2 | **3.5** | Med |
| 9 | Images load out of priority order | 8 | 1 | 0.8 | 2 | **3.2** | Med |
| 10 | Page unusable while loading (TBT) | 9 | 2 | 0.7 | 4 | **3.2** | Med |
| 11 | Controls/images invisible to assistive tech | 3 | 2 | 1.0 | 2 | **3.0** | Med |
| 12 | Consent wall looks broken on first load | 7 | 2 | 0.6 | 3 | **2.8** | Med |
| 13 | Excess composite layers + GPU hacks | 7 | 1 | 0.8 | 2 | **2.8** | Med |
| 14 | One visit downloads 34 MB | 9 | 2 | 0.6 | 5 | **2.2** | Med |
| 15 | First-party JS ships as one un-split bundle | 10 | 1 | 0.6 | 4 | **1.5** | Low |
| 16 | Repeat visits barely cache | 5 | 1 | 0.6 | 4 | **0.8** | Low |

Two things the ranking makes explicit. The single highest-ROI fix is trivial —
`fetchpriority` + preload on the lead image scores 24 because it's a one-line
change straight at the failing metric. And the ad-stack cut (row 9) scores *low*
not because it doesn't matter — it's the root of the byte weight — but because
RICE divides by effort and that fix is an org-level, revenue-touching program.
Read the low-scorers as "high value, expensive," not "skip."

## Rendering

- The page is blank for several seconds before anything renders.
  - **Baseline**: FCP (6.3 s mobile lab). TTFB is fine at 0.2 s, so it isn't the server.
  - **Cause**:
    - A 105.6 KiB stylesheet (`All.min.css`) and a 41.5 KiB first-party script (`script.js`) are render-blocking.
    - Parse.ly and the OneTrust consent script (`OtAutoBlock.js`) also sit in the critical path (PSI estimates 740 ms).
  - **Solution**:
    - Inline the critical CSS and load the rest non-blocking.
    - Split and defer `script.js`; load Parse.ly and the consent script without blocking render.
  - **Priority (RICE)**: R 10 × I 2 × C 0.8 ÷ E 3 = **5.3** (High).

- The main article image shows up late.
  - **Baseline**: LCP (38.6 s mobile lab / 2.9 s field — the field assessment fails on LCP).
  - **Cause**:
    - The lead promo image (`div.PagePromo-media img`, `dims.apnews.com`, 725×485) has no `fetchpriority`, so it competes with ~2,300 other requests instead of loading first.
    - It also sits behind the render-blocking chain above.
  - **Solution**:
    - Add `fetchpriority="high"` and a preload to the lead promo image.
  - **Priority (RICE)**: R 10 × I 3 × C 0.8 ÷ E 1 = **24.0** (High — the top quick win: one attribute straight at the failing LCP).

- The page can't be used while it's still loading.
  - **Baseline**: TBT (1,250 ms mobile / 4,650 ms desktop lab); main-thread work 21.2 s, JS execution 5.9 s.
  - **Cause**:
    - Third-party ad and tracking scripts saturate the main thread — Rubicon 776 ms, Google Tag Manager 575 ms, Web Content Assessor 524 ms, Quantcast 491 ms, pub.network 423 ms.
  - **Solution**:
    - Load ad and analytics scripts after first paint, lazy-load below-fold ad slots, and break up the long tasks.
  - **Priority (RICE)**: R 9 × I 2 × C 0.7 ÷ E 4 = **3.2** (Med).

- Images don't load in the order the reader needs.
  - **Baseline**: Performance / LCP.
  - **Cause**:
    - Only 2 of 106 images set a priority, so the browser can't tell the lead story image from ad creatives and below-fold photos — ads and secondary images often download before the headline image.
  - **Solution**:
    - `fetchpriority="high"` on the lead image and `fetchpriority="low"` / `loading="lazy"` on the rest.
  - **Priority (RICE)**: R 8 × I 1 × C 0.8 ÷ E 2 = **3.2** (Med).

- The consent wall makes the page look broken on first load.
  - **Baseline**: LCP / perceived load.
  - **Cause**:
    - The OneTrust consent modal ("603 partners") covers the article on the first mobile load; its script is render-blocking (900 ms) and the ad stack waits on the consent choice, so the top of the page sits empty until it resolves.
  - **Solution**:
    - Paint the article underneath the modal immediately, load the consent script without blocking render, and hold non-essential vendors until after the user chooses.
  - **Priority (RICE)**: R 7 × I 2 × C 0.6 ÷ E 3 = **2.8** (Med).

## Networking

### Good

- Text compression is effective.
  - **Baseline**: network compression.
  - JS and CSS ship br/gzip — 37.0 MB uncompressed comes down as 7.8 MB (~79 %). Images aren't gzipped, which is correct: they're already compressed formats.

- First-party caching is effective.
  - **Baseline**: network caching.
  - Static assets are content-hashed with a long TTL, so a soft refresh drops transfer from 34.3 MB to 7.2 MB (~79 % less over the wire).

### Corrective

- One visit downloads far too much.
  - **Baseline**: total transfer 34.3 MB across 2,338 requests — expensive on a phone plan and slow on a weak connection.
  - **Cause**:
    - The ad/bidding/tracking stack — Rubicon 385 KiB, Google Tag Manager 330 KiB, and ~20 more vendors.
    - Images account for 15.7 MB of the total.
  - **Solution**:
    - Cut the number of ad partners and bidders, lazy-load below-fold ad slots, and defer non-critical vendor scripts.
  - **Priority (RICE)**: R 9 × I 2 × C 0.6 ÷ E 5 = **2.2** (Med — high value, but an org-level, revenue-touching program, so RICE ranks the ROI low).

- Repeat visits barely get faster.
  - **Baseline**: repeat-view transfer 7.2 MB (vs 34.3 MB fresh).
  - **Cause**:
    - First-party assets cache well, but the 2,000+ third-party ad/tracker requests are uncacheable and re-download on every visit.
  - **Solution**:
    - Caching can't fix this — reduce the ad/tracker calls themselves (fewer vendors, lazy-load, consolidate).
  - **Priority (RICE)**: R 5 × I 1 × C 0.6 ÷ E 4 = **0.8** (Low).

## Build output

Findings from the shipped build outputs (Day 7) — how the JS, CSS and image assets
are bundled. See the [baseline](baseline.md) bundle analysis. (Source maps are
handled correctly — not exposed in production — so that's not a finding here.)

- The CSS ships as one ~770 KB site-wide bundle, mostly unused per page.
  - **Baseline**: a single `All.min.css` — ~770 KB uncompressed (105.6 KiB brotli), render-blocking on every page; DevTools Coverage measures **88% unused** on the homepage (694 KB of 788 KB).
  - **Cause**:
    - No per-route or per-component CSS splitting: the whole design system (article, gallery, search and donate templates) ships on the homepage, where almost none of it applies.
  - **Solution**:
    - Extract the critical CSS the page needs and load the rest non-blocking; split the sheet per route/component so a page only ships what it renders.
  - **Priority (RICE)**: R 10 × I 2 × C 0.7 ÷ E 3 = **4.7** (High).

- Responsive images set `srcset` but no `sizes`.
  - **Baseline**: 68 of 100 `<img>` carry `srcset`; 0 set a `sizes` attribute.
  - **Cause**:
    - With no `sizes`, the browser assumes each image is 100vw and picks a larger `srcset` candidate than the slot needs — over-fetching bytes the layout never uses.
  - **Solution**:
    - Add a correct `sizes` to each responsive image so the browser can choose the smallest candidate that fits the slot.
  - **Priority (RICE)**: R 9 × I 1 × C 0.9 ÷ E 2 = **4.1** (High).

- First-party JavaScript ships as one un-split bundle.
  - **Baseline**: a single `All.min.js` (~450 KB uncompressed, **84% unused** per Coverage — 381 KB of 452 KB) plus a blocking `apcdp.apnews.com/script.js`; no route or component splitting.
  - **Cause**:
    - Everything is concatenated into one "All" bundle, so the homepage parses and executes code for templates it never renders. Tree-shaking can't help across a deliberately-concatenated file.
  - **Solution**:
    - Split by route and lazy-load heavy or rarely-used components on interaction/visibility, so each page ships only the JS it needs.
  - **Priority (RICE)**: R 10 × I 1 × C 0.6 ÷ E 4 = **1.5** (Low).

## Frames and layers

Day 8 pass on the rendering pipeline — critical CSS, the flame chart, and
compositing. Measured live; see the baseline's rendering/frames/layers section.

- Critical CSS isn't really extracted — the full stylesheet still blocks first paint.
  - **Baseline**: the head inlines ~145 KB across 19 `<style>` blocks, yet the 801 KB `All.min.css` (88% unused) still loads render-blocking; the trace estimates ~1,166 ms of FCP waiting on it.
  - **Cause**:
    - The site pays for inlined CSS *and* a render-blocking full sheet — the inline blocks don't cleanly cover the fold and the big sheet is never deferred.
  - **Solution**:
    - Inline only the above-the-fold critical CSS and load the rest non-blocking (preload + swap); split the sheet so a page ships what it renders.
  - **Priority (RICE)**: R 10 × I 2 × C 0.6 ÷ E 3 = **4.0** (High).

- Scrolling drops frames — about 28 fps under load.
  - **Baseline**: a scripted scroll on a 4× CPU profile holds only ~28 fps — 41% of frames miss the 16.7 ms budget, 20% exceed 33 ms, and the worst frame stalls 376 ms.
  - **Cause**:
    - Scroll-linked forced reflows (first-party `updateScrollShades`, Primis `checkResize`) read layout after DOM writes; a 9,569-node DOM makes each reflow expensive; ~48 ad iframes repaint as they scroll into view. A first-party Puzzmo handler also forces 180 ms of layout at load.
  - **Solution**:
    - Batch layout reads before writes to stop the thrashing, cut the DOM size and lazy-mount below-fold ad slots, and throttle scroll handlers to `requestAnimationFrame`.
  - **Priority (RICE)**: R 8 × I 2 × C 0.7 ÷ E 3 = **3.7** (Med).

- Too many composite layers, plus needless GPU hacks.
  - **Baseline**: ~48 ad iframes/videos each promote a layer, alongside 5 fixed / 3 sticky / 31 transformed elements; `All.min.css` also carries 3 `translateZ(0)` hacks and 2 `will-change` rules.
  - **Cause**:
    - Each layer costs GPU memory and excess layers slow compositing itself; the `translateZ(0)` / `will-change` promotions are always-on and buy nothing when nothing is animating.
  - **Solution**:
    - Drop the `translateZ(0)` hacks and apply `will-change` only just before an animation (remove it on `transitionend`); reduce the number of simultaneously-mounted ad iframes.
  - **Priority (RICE)**: R 7 × I 1 × C 0.8 ÷ E 2 = **2.8** (Med).

## Mobile

Mobile is Google's primary ranking signal, and Slow 4G + 4× CPU is the condition
most of AP's audience actually browses under. These findings only appear, or only
matter, on mobile.

### Corrective

- On a real mobile connection the page barely loads.
  - **Baseline**: throttled-mobile lab (Slow 4G + 4× CPU). PSI mobile LCP 38.6 s, FCP 6.3 s, Speed Index 17.1 s; a local applied-throttle run agrees (LCP 42.3 s, FCP 7.2 s, SI 25.8 s) and is so slow that Lighthouse can't finish a scored run (TBT returns `NO_TTI_CPU_IDLE_PERIOD` — the main thread never goes idle). Desktop lab LCP is 7.1 s — about 6× faster on the same page.
  - **Cause**:
    - The render-blocking chain and the 20.4 MB fresh payload are tolerable on a fast desktop link but collapse on a slow mobile radio with a 4× slower CPU: the lead image is gated behind the whole download, and the main thread never clears.
  - **Solution**:
    - The rendering and networking fixes elsewhere in this report (inline critical CSS, defer JS, `fetchpriority` on the lead image, cut ad bytes and requests) pay off most here — budget and test against Slow 4G + 4× CPU, not a desktop link.
  - **Priority (RICE)**: R 10 × I 3 × C 0.7 ÷ E 5 = **4.2** (High — an umbrella finding; its effort spans the fixes above, so it's scored as the mobile program overall).

- Tap targets are too small and too close together.
  - **Baseline**: the Lighthouse accessibility audit flags 49 undersized or crowded touch targets. Mobile-only: a mouse pointer doesn't care, a thumb does.
  - **Cause**:
    - Nav links, share buttons and article-card links are sized and spaced for a cursor, so on a phone they fall under the ~24 px minimum and are easy to mis-tap.
  - **Solution**:
    - Give tap targets at least 24×24 CSS px (48×48 recommended) with enough spacing at the mobile breakpoint.
  - **Priority (RICE)**: R 7 × I 1 × C 1.0 ÷ E 2 = **3.5** (Med).

### Good

- The layout stays stable on mobile.
  - **Baseline**: CLS 0.06 (field) / 0.064 (local applied throttle) / 0.001 (PSI) — all under the 0.1 threshold.
  - Despite the ad slots and lazy-loaded images, content doesn't jump around as it loads on a phone. CLS is the one Core Web Vital AP already passes on mobile — the mobile problem is load speed, not layout instability.

## Accessibility

- Interactive controls and images are invisible to assistive technology.
  - **Baseline**: PSI Accessibility 75 (mobile) / 79 (desktop); a Lighthouse accessibility audit flags eight issue types (the undersized-tap-target one is in the Mobile section above).
  - **Cause**:
    - 2 buttons and 2 links have no accessible name; 4 images have no `alt`; 1 iframe has no title.
    - 29 `aria-hidden` elements contain focusable descendants; headings aren't in sequential order; one text/background pair fails contrast.
  - **Solution**:
    - Add `aria-label` to icon-only buttons and links, `alt` to images, and a `title` to the iframe.
    - Remove focusable children from `aria-hidden` regions, fix the heading order, and fix the contrast pair.
  - **Priority (RICE)**: R 3 × I 2 × C 1.0 ÷ E 2 = **3.0** (Med).
