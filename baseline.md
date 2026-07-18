# Baseline

Primary page: https://apnews.com/. Core Web Vitals and PageSpeed Insights
captured 3 July 2026 (Lighthouse 13.4.0); networking 5 July 2026, the local
applied-throttle Lighthouse runs 8 and 11 July 2026, and the WebPageTest run
11 July 2026. Screenshots in `screenshots/`.

## Core Web Vitals

Field data (CrUX, real users, 75th percentile). Assessment: **Failed** on both —
LCP is above the 2.0 s "good" threshold (tightened March 2026); INP and CLS pass.

### Desktop

- **Largest Contentful Paint (LCP)**: 3.6 s
- **Interaction to Next Paint (INP)**: 155 ms
- **Cumulative Layout Shift (CLS)**: 0.03
- **First Contentful Paint (FCP)**: 1.9 s
- **Time to First Byte (TTFB)**: 0.2 s

### Mobile

- **Largest Contentful Paint (LCP)**: 2.9 s
- **Interaction to Next Paint (INP)**: 178 ms
- **Cumulative Layout Shift (CLS)**: 0.06
- **First Contentful Paint (FCP)**: 2.0 s
- **Time to First Byte (TTFB)**: 0.2 s

## PageSpeed Insights at `/`

Lab data (Lighthouse). Mobile is throttled to the class setting — **Slow 4G with
a 4× CPU slowdown, applied (not simulated)** — on an emulated mid-range phone
(Moto G Power); desktop uses the emulated desktop profile with no throttling.
Mobile is the primary signal (mobile-first indexing), so the mobile column is the
one the findings track.

### Desktop

- **Performance**: 33
- **Accessibility**: 79
- **Best Practices**: 54
- **SEO**: 85
- **Agentic Browsing**: 1/2

Metrics: FCP 1.3 s · LCP 7.1 s · TBT 4,650 ms · CLS 0 · Speed Index 10.9 s

### Mobile

- **Performance**: 31
- **Accessibility**: 75
- **Best Practices**: 54
- **SEO**: 85
- **Agentic Browsing**: 1/2

Metrics: FCP 6.3 s · LCP 38.6 s · TBT 1,250 ms · CLS 0.001 · Speed Index 17.1 s

Worst on mobile: LCP 38.6 s, Speed Index 17.1 s, FCP 6.3 s, TBT 1,250 ms.

## Local Lighthouse — Applied throttling, median of 3 (mobile)

Step 2 of the class method: local Lighthouse in DevTools with **applied** throttling
(Slow 4G + 4× CPU, "Simulated throttling" off), run **three times** with the median
reported. Screenshots in `screenshots/04-mobile-throttling/`.

| Metric | Run 1 | Run 2 | Run 3 | Median |
|--------|:-----:|:-----:|:-----:|:------:|
| Performance | error* | 33 | 33 | **33** |
| FCP | 7.2 s | 13.0 s | 19.5 s | **13.0 s** |
| LCP | 42.3 s | 40.2 s | 52.2 s | **42.3 s** |
| TBT | error* | 1,030 ms | 1,030 ms | **1,030 ms** |
| CLS | 0.064 | 0.001 | 0 | **0.001** |
| Speed Index | 25.8 s | 18.9 s | 28.7 s | **25.8 s** |

*Run 1 timed out before Lighthouse could compute TBT or the score
(`NO_TTI_CPU_IDLE_PERIOD` — the main thread never went idle).

The spread is the point: across three identical runs FCP swings 7–20 s and LCP
40–52 s, which is exactly why the class takes the median of 3 rather than trusting
one local run. The median (LCP 42 s, FCP 13 s, TBT 1,030 ms) sits alongside PSI
mobile (LCP 38.6 s, TBT 1,250 ms) and agrees — the throttled-mobile load is
catastrophic. Runs 2–3 also scored Accessibility 71, Best Practices 77, SEO 85.
Caveat: the runs weren't in a fresh Incognito profile (Lighthouse flagged stored
IndexedDB data), so PSI stays the primary number.

## WebPageTest — filmstrip (mobile)

Step 3 of the class method — a WebPageTest run for the waterfall and filmstrip, to
see when the page becomes usable. Captured 11 July 2026; screenshots in
`screenshots/06-webpagetest/`.

- **Config**: iPhone 15, Chrome v145, **4G (9 Mbps, 170 ms RTT)**, Tokyo — a faster
  link than the Slow-4G lab, so these times run lower than PSI/Lighthouse.
- **Result**: a **Page Load Timeout** — the page never finished inside WebPageTest's
  budget (Total Time 30.4 s, 537 requests, 8 MB first view).
- **Metrics**: FCP 3.39 s · LCP 3.39 s · Start Render 3.4 s · Speed Index 8.85 s ·
  TBT 3.25 s · CLS 0.005 · TTFB 0.56 s.
- **When does it become usable?** The filmstrip is blank through ~3.4 s (start
  render) — nothing is visible for the first three-plus seconds — then it fills in
  but keeps loading past 30 s. WebPageTest's own read: *"8 render-blocking requests…
  took a long time to become interactive… many render-blocking 3rd-party requests
  that could be a single point of failure."*
- **Field (CrUX, 12 Jun – 9 Jul 2026)** shown alongside: FCP 2.22 s, LCP 2.90 s,
  CLS 0.04, TTFB 0.24 s, INP 0.19 s — real users beat every lab profile, but LCP
  still lands in "needs improvement."

## Network Activity

DevTools Network panel — fresh load, then a soft refresh. Transfer size first,
uncompressed resource size in parentheses. These are byte and request counts, so
they don't change with throttling; the mobile *timing* under the class throttle is
in the two sections above.

- **protocol**: http/2 and http/3 (h3 via Alt-Svc; a few http/1.1)
- **caching**
  - first-party static assets: content-hashed, long TTL (served from cache on refresh)
  - third-party ad/tracker calls: uncacheable, re-fetched every load
- **compression**
  - text (HTML/JS/CSS): br/gzip
  - images/binary: none (already-compressed formats)

### Desktop

- **requests**: 2,338
- **fresh**: 34.3 MB (87.2 MB)
  - **js/css**: 7.8 MB (37.0 MB)
  - **images**: 15.7 MB (20.8 MB)
- **refresh**: 7.2 MB (78.9 MB)

### Mobile

- **requests**: 3,169
- **fresh**: 20.4 MB (86.4 MB)
  - **js/css**: 8.2 MB (43.6 MB)
  - **images**: 8.4 MB (9.6 MB)
- **refresh**: 7.2 MB (75.1 MB)

Mobile ships less than desktop (20.4 MB vs 34.3 MB) — mostly smaller responsive
images — but it's still a lot for a phone, and the third-party request count
barely drops.

## Diagnostics (mobile, PSI Insights)

Where the time and bytes go — the causes `findings.md` builds on. Screenshots in
`screenshots/02-baseline-findings/`.

- Render-blocking requests — est. 740 ms (`All.min.css` 105.6 KiB / 4,800 ms;
  first-party `script.js` 41.5 KiB / 2,850 ms; Parse.ly 1,310 ms; OneTrust
  `OtAutoBlock.js` 900 ms).
- LCP element — the lead promo image (`div.PagePromo-media img`,
  `dims.apnews.com`, 725×485): discoverable and not lazy-loaded, but no
  `fetchpriority="high"`.
- Minimize main-thread work — 21.2 s. Reduce JavaScript execution time — 5.9 s.
- Third parties — Rubicon Project 385 KiB (591 ms), Google Tag Manager 330 KiB
  (446 ms), then ~20 more ad/analytics vendors.
- Consent — OneTrust/Optanon ("603 partners"): render-blocking `OtAutoBlock.js`
  (900 ms) plus 395 ms of main-thread time, and a full-screen modal on first load.
- Accessibility (Lighthouse a11y audit) — 8 failing checks: buttons/links without
  accessible names (2 + 2), images without `alt` (4), iframe without title (1),
  `aria-hidden` regions with focusable descendants (29), heading order, 49
  undersized touch targets, and one contrast failure.

## Bundle analysis

How the build outputs are shipped (Day 7 method). Inspected live on the homepage
with DevTools on 8 July 2026; third-party costs are from the PSI mobile report, and
unused-code figures from the DevTools Coverage tab (screenshots in
`screenshots/05-bundles/`). Coverage on a fresh load shows the page executes only
**38% of the 35.2 MB** of JS and CSS it pulls — **21.8 MB downloaded and never
used**.

### JavaScript

- **Bundling** — first-party JS ships as a single concatenated bundle,
  `All.min.<hash>.gz.js` (~450 KB uncompressed), content-hashed and pre-gzipped,
  plus a deferred `webcomponents-loader` polyfill and a blocking
  `apcdp.apnews.com/script.js` (consent/data-platform, no `async`/`defer`). There's
  no route- or component-level splitting — the homepage loads the same "All" bundle
  every page would.
- **Unused JS** — Coverage measures **84% of the bundle unused** on the homepage
  (381 KB of 452 KB); a single site-wide bundle ships code for templates it never
  renders, and tree-shaking can't remove what's deliberately concatenated into one
  file.
- **Source maps** — none exposed: `All.min.js.map` returns 404 and the bundle has
  no `sourceMappingURL`. Correct for production (Day 7: maps shouldn't be public).

### CSS

- **Bundling** — same pattern: one `All.min.<hash>.gz.css`, **~770 KB uncompressed**
  (105.6 KiB brotli). Single site-wide stylesheet, no per-route/component splitting,
  and render-blocking.
- **Unused CSS** — Coverage measures **88% of the bundle unused** on the homepage
  (694 KB of 788 KB): most selectors target the article, gallery, search and donate
  templates that aren't on this page.
- Third-party CSS adds 7 more stylesheets (Google Fonts ×2, Bounce Exchange,
  Viafoura ×3, Primis).

### Images

- **Sizes/formats** — images run through the `dims.apnews.com` proxy, which outputs
  **WebP at quality 90** and resizes per request, so format is modern and variants
  are generated at serve time. 68 of 100 `<img>` carry `srcset` and 56 are
  `loading="lazy"`.
- **Missing `sizes`** — 0 of those 68 responsive images set a `sizes` attribute, so
  the browser assumes the image is 100vw and tends to pick a larger `srcset`
  candidate than the slot needs.
- **Full-resolution is exposed** — the proxy URL carries `?url=<original>` pointing
  straight at the multi-megapixel source on `assets.apnews.com` (crops reveal
  6000×4000 and 5870×3913 originals), and the `resize/` path segment is editable, so
  any size — including the full-res original — can be requested as WebP.

### Third-party resources

- **Inventory** — ~46 distinct third-party domains on the homepage (800+ script
  requests over the full load). Almost all are ads / real-time bidding / identity /
  data-management: Google (GPT, DoubleClick, GTM, IMA), Amazon APS, Criteo,
  pub.network, Bounce Exchange, Dianomi, Connatix, Primis, JW Player, ID5,
  Permutive, Lotame (`crwdcntrl`), BlueConic, Nativo (`postrelease`), Parse.ly,
  comScore, Sailthru, Kameleoon, OneSignal, Viafoura, Zephr — plus obscure ones with
  no obvious owner (`usablenet`, `kidpowers`, `wknd.ai`, `tru.am`, `riverdrop`,
  `rapidedge`, `html-load.cc`).
- **Loading** — most load `async`, but several load synchronously and block render
  (Primis, Dianomi, a Parse.ly experiments script, Riverdrop), and the first-party
  `script.js` is blocking too.
- **Impact** (PSI mobile) — Rubicon 385 KiB / 591 ms and Google Tag Manager 330 KiB /
  446 ms are the heaviest by transfer; Web Content Assessor (524 ms), Quantcast
  (491 ms) and pub.network (423 ms) are the heaviest on the main thread, across
  ~20+ more vendors. This is the bulk of both the byte weight and the blocking.
- **Unused at load** (Coverage) — the biggest single blocks of dead code are all
  third-party video/ad scripts, pulled in full but barely run on the homepage:
  ntv.io `load.js` (729 KB unused), Google reCAPTCHA (598 KB), Primis HLS (496 KB)
  and sekindo `liveVideo` (482 KB).

## Rendering, frames and layers

Day 8 pass — DevTools Coverage, the Performance flame chart and the Layers panel,
captured 11 July 2026. Screenshots in `screenshots/07-frames-layers/`.

### Critical CSS

- The `<head>` carries a few small inline `<style>` blocks (a Viafoura font-size
  override, `:root { --button-border-radius }`) and a font `preload`, but no real
  above-the-fold critical CSS — the head is dominated by render-blocking third-party
  scripts loaded up front, and the main `All.min.css` still loads. So critical CSS
  isn't meaningfully extracted.
- Coverage this run: **47% of 17.4 MB of JS/CSS used, 9.3 MB unused**. The first-party
  CSS (`All.min…css`) is **90% unused** (721 KB of 801 KB) and the first-party JS
  (`All.min…js`) is **83% unused** (378 KB of 456 KB) — all still parsed each load. The
  biggest unused blocks are third-party video/ad/consent scripts: ntv.io (729 KB
  unused), reCAPTCHA (589 KB), Primis video (496 + 426 KB), sekindo (469 KB), pubfig,
  Viafoura, DoubleClick, Bounce Exchange, OneTrust, GTM.

### Flame chart — frames

- The Frames track shows the page **stall for whole seconds** during load — a
  **1,733 ms** frame and then a **5,058 ms** frame (long red frames = dropped). Over
  the ~49 s load the main thread spends **14.5 s scripting, 4.7 s painting, 2.1 s
  rendering**. Almost all of it is third-party: `[unattributed]` 23 s, ftstatic 2.2 s,
  ad-score 0.8 s, DoubleVerify 0.4 s — with LongTail Ad Solutions alone shipping
  **46 MB** of video; first-party `apnews.com` is only ~0.3 s.
- So frames aren't just occasionally dropped — the main thread is saturated by
  ad/video scripting and the page freezes for multiple seconds at a time. For a news
  homepage that's excessive and directly hurts responsiveness.

### Layers and animations

- **Paint layers** — the Layers panel shows **20 composite layers using 814 MB** of
  GPU memory. The document rootScroller alone is **383 MB** (compositing reason: "Is
  the document.rootScroller… a scrollable overflow element using accelerated
  scrolling"), and it carries **slow-scroll regions** — full-page touch and wheel event
  handlers — which force non-passive (slow) scrolling. Other layers: a video
  (1096×618), an iframe (600×500), the sticky header (`.Page-header-stickyWrap`), three
  Bounce Exchange scroll-shades and the usablenet accessibility widget (`usntA40*`).
- **What forces them** — the sampled layers are promoted for legitimate reasons
  (rootScroller/accelerated scrolling, video, iframe, sticky), not obvious
  `will-change`/`translateZ` hacks; the cost is the sheer **volume (20 layers, 814 MB)**
  plus the slow-scroll event handlers. Animations: two ran during load (the purple
  bars), on composite-friendly properties — so the jank is the scripting and layer
  weight, not the animations themselves.

## Rendering strategy

Day 12 pass — the three-check fingerprint (view source, then reload with
JavaScript disabled, then read the document response in the Network panel), run
per page type rather than once for the whole site. Screenshots in
`screenshots/08-rendering-strategies/`.

### What's in use, per page

- **Homepage (`/`)** — **server-side rendered**, edge-cached for a short window.
  The document arrives with the content already in the HTML (29 headings and every
  story link are in view-source, not injected by JS), the server header is
  `x-powered-by: Brightspot` (AP runs on the Brightspot Java CMS, not a JS
  framework), and the markup carries 926 `bsp-` attributes from that CMS. Cloudflare
  fronts it with `cache-control: public, max-age=120` and returns `cf-cache-status:
  HIT` — a **120-second micro-cache**. The origin render isn't cheap:
  `x-envoy-upstream-service-time` runs ~1.4–2.6 s, which is exactly why the
  micro-cache exists — you don't want to pay that render on every request.
- **Article (`/article/…`)** — the **same SSR engine, but cached at the edge for a
  year**. The article response carries `cache-control: public, max-age=30,
  s-maxage=31536000` with `cf-cache-status: HIT`: the browser revalidates after 30 s
  but the CDN keeps the rendered HTML for up to a year. In effect an article is
  **static-at-the-edge** — rendered once by Brightspot, then served like SSG with
  stale-while-revalidate. The full article body (201 `<p>`) is in the document.
- **No hydration framework.** There's no `__NEXT_DATA__`, React root, Fusion or
  serialized-state blob — nothing to hydrate. The client JS (`All.min.js` plus the
  ~46 third-party scripts) is progressive enhancement and ad/embed code layered on
  top of already-complete HTML, so the ~1 s TBT is that layer, not a hydration cost.
- **Client-rendered islands inside the SSR page.** A few regions are left as empty
  placeholders for JS to fill: `#riverdrop-widget` is an empty
  `<div style="min-height:600px">` populated by a third-party quiz embed
  (`client.riverdrop.com`), and `#desktop-entitlements` (the Zephr paywall) is
  filled client-side. Both appear on the homepage and the article.

### How it affects users, and the tradeoffs

The SSR base is why the server-dependent field numbers are fine: TTFB 0.2 s and FCP
1.9–2.0 s — content is in the first response, so the browser paints real text early
instead of waiting on a bundle. The metric signature matches "SSR with a heavy
client layer," not CSR: fast paint, then a lag before the page is interactive
(field INP 155–178 ms, borderline; lab TBT 1,030–4,650 ms) — the uncanny-valley
pattern, here caused by ad/embed JS rather than hydration. The edge caching is the
other win: the expensive Brightspot render is paid once per 120 s (homepage) or once
per publish (article), not per visit.

The tradeoff is the CSR islands. The 600 px `riverdrop` region and the paywall
region are blank until their JS runs, so on a slow or blocked connection those slots
sit empty while the SSR content around them is instant — and they depend on
third-party JS for content that could be server-rendered.

### Is it a good choice for those pages — and the best one?

A good choice, and for these two page types about as close to the best as it gets.
An article is the same for everyone and changes rarely
after publish, so SSR-once + long edge cache (static-at-the-edge) is the textbook
fit; the homepage changes with the news cycle, so a short micro-cache over SSR is
the honest freshness call. This is a per-route strategy done well — so the fix isn't
to change the strategy (the LCP/TBT problems would survive a migration; they're
image, bundle and ad-stack problems), it's to stop the client layer and the CSR
islands from spending the SSR head start. See the rendering-strategy findings.

## Note on method

Scores are from PageSpeed Insights, which runs Lighthouse on Google's
infrastructure — a clean run without local-machine noise, and it applies the same
Slow 4G + 4× CPU mobile throttle the class specifies. Mobile is weighted as
primary (mobile-first indexing); since Lighthouse can't measure INP, TBT is the
lab proxy. The local Lighthouse and WebPageTest sections are the class's Steps 2 and 3 — an
applied-throttle median of 3, then a filmstrip run; they corroborate PSI, though
the local runs weren't in a clean Incognito profile, so PSI stays the primary
number. Diagnostics come from the same PSI
report's Insights/Diagnostics; the accessibility failures are from a separate
Lighthouse accessibility audit of the page.
