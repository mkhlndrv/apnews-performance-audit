# Baseline

Primary page: https://apnews.com/ — captured 3 July 2026, PageSpeed Insights
(Lighthouse 13.4.0), mobile is an emulated Moto G Power on Slow 4G. Score
screenshots in `screenshots/`.

## Core Web Vitals

Field data (CrUX, real users, 75th percentile, mobile). Assessment: **Failed** —
LCP (2.9 s) is above the 2.0 s "good" threshold (tightened March 2026).

- **Largest Contentful Paint (LCP)**: 2.9 s
- **Interaction to Next Paint (INP)**: 178 ms
- **Cumulative Layout Shift (CLS)**: 0.06

## PageSpeed Insights at `/` (mobile)

- **Performance**: 31
- **Accessibility**: 75
- **Best Practices**: 54
- **SEO**: 85

- **First Contentful Paint**: 6.3 s
- **Largest Contentful Paint**: 38.6 s
- **Total Blocking Time**: 1,250 ms
- **Cumulative Layout Shift**: 0.001
- **Speed Index**: 17.1 s

Desktop for comparison: Performance 33, LCP 7.1 s, TBT 4,650 ms, Speed Index
10.9 s; field CWV also Failed (LCP 3.6 s).

Worst metrics on mobile, in order: LCP 38.6 s, Speed Index 17.1 s, FCP 6.3 s,
TBT 1,250 ms. CLS is fine.

## Diagnostics

The scores above are from PageSpeed Insights, which runs Lighthouse on Google's
infrastructure — a clean run without local-machine noise. Mobile is the primary
target (mobile-first indexing), and since Lighthouse can't measure INP, TBT is
the lab proxy. The causes come from the same report's Insights and Diagnostics
for the mobile run (screenshots in `screenshots/`).

- Render-blocking requests — est. 740 ms. `All.min.css` 105.6 KiB (4,800 ms) and
  first-party `script.js` 41.5 KiB (2,850 ms), plus Parse.ly (1,310 ms) and the
  OneTrust consent script (`OtAutoBlock.js`, 900 ms).
- LCP element — the lead promo image (`div.PagePromo-media img`,
  `dims.apnews.com`, 725×485): discoverable and not lazy-loaded, but no
  `fetchpriority="high"`.
- Minimize main-thread work — 21.2 s. Reduce JavaScript execution time — 5.9 s.
- Third parties — Rubicon Project 385 KiB (591 ms), Google Tag Manager 330 KiB
  (446 ms), then ~20 more ad/analytics vendors (Web Content Assessor, Quantcast,
  pub.network, Viafoura, Kameleoon, Amazon Ads, Doubleclick, Wunderkind,
  permutive, …).
- Consent — OneTrust/Optanon ("603 partners"): render-blocking `OtAutoBlock.js`
  (900 ms) plus 395 ms of main-thread time, and a full-screen modal on first load.

## Network Activity

Captured 5 July 2026 from the DevTools Network panel (desktop, no throttling):
fresh load, then a soft refresh. Transfer size first, uncompressed resource size
in parentheses.

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
