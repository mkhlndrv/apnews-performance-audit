# Baseline

Primary page: https://apnews.com/. Core Web Vitals and PageSpeed Insights
captured 3 July 2026 (Lighthouse 13.4.0); networking captured 5 July 2026 and the
local applied-throttle Lighthouse run 8 July 2026, both from DevTools. Screenshots
in `screenshots/`.

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

## Local Lighthouse — Applied throttling (mobile)

Step 2 of the class method: a local Lighthouse run in DevTools with **applied**
throttling (the "DevTools throttling" option — Slow 4G + 4× CPU, not simulated),
to sit next to the PSI mobile column and check it against a second setup.
Screenshots in `screenshots/04-mobile-throttling/`.

- **FCP**: 7.2 s
- **LCP**: 42.3 s
- **Speed Index**: 25.8 s
- **CLS**: 0.064
- **TBT**: did not compute — Lighthouse returned `NO_TTI_CPU_IDLE_PERIOD` (the
  main thread never went idle), and the overall Performance score errored because
  "the page loaded too slowly to finish within the time limit."

This lines up with PSI mobile (LCP 42.3 s vs 38.6 s, FCP 7.2 s vs 6.3 s) and goes
one step further: under the real class throttle the page is so slow that
Lighthouse can't finish a scored run at all. Two caveats — the run wasn't in a
fresh Incognito profile (Lighthouse warned about stored IndexedDB data) and it
timed out before finishing, so it's corroboration of PSI, not a clean
median-of-3.

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
with DevTools on 8 July 2026; third-party costs are from the PSI mobile report.

### JavaScript

- **Bundling** — first-party JS ships as a single concatenated bundle,
  `All.min.<hash>.gz.js` (~450 KB uncompressed), content-hashed and pre-gzipped,
  plus a deferred `webcomponents-loader` polyfill and a blocking
  `apcdp.apnews.com/script.js` (consent/data-platform, no `async`/`defer`). There's
  no route- or component-level splitting — the homepage loads the same "All" bundle
  every page would.
- **Unused JS** — a single site-wide bundle means the homepage ships code for
  templates it never renders. Tree-shaking can't remove what's deliberately
  concatenated into one file.
- **Source maps** — none exposed: `All.min.js.map` returns 404 and the bundle has
  no `sourceMappingURL`. Correct for production (Day 7: maps shouldn't be public).

### CSS

- **Bundling** — same pattern: one `All.min.<hash>.gz.css`, **~770 KB uncompressed**
  (105.6 KiB brotli). Single site-wide stylesheet, no per-route/component splitting,
  and render-blocking.
- **Unused CSS** — a 770 KB global sheet on the homepage is nearly all unused; most
  selectors target the article, gallery, search and donate templates that aren't on
  this page.
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

## Note on method

Scores are from PageSpeed Insights, which runs Lighthouse on Google's
infrastructure — a clean run without local-machine noise, and it applies the same
Slow 4G + 4× CPU mobile throttle the class specifies. Mobile is weighted as
primary (mobile-first indexing); since Lighthouse can't measure INP, TBT is the
lab proxy. The local Lighthouse section is the class's "applied, not simulated"
throttle run on my own machine; it corroborates PSI but wasn't a clean Incognito
median-of-3, so PSI stays the primary number. Diagnostics come from the same PSI
report's Insights/Diagnostics; the accessibility failures are from a separate
Lighthouse accessibility audit of the page.
