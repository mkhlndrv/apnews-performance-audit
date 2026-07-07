# Findings

Grouped by area, corrective findings first within each. Each finding is a
distinct, independently observable problem — where several causes feed the same
symptom, they sit under one finding. Rendering numbers come from the PSI mobile
Insights/Diagnostics, networking from the DevTools Network panel, mobile from the
throttled-mobile runs (Slow 4G + 4× CPU), and accessibility from a Lighthouse
accessibility audit. See [baseline](baseline.md).

## Rendering

- The page is blank for several seconds before anything renders.
  - **Baseline**: FCP (6.3 s mobile lab). TTFB is fine at 0.2 s, so it isn't the server.
  - **Cause**:
    - A 105.6 KiB stylesheet (`All.min.css`) and a 41.5 KiB first-party script (`script.js`) are render-blocking.
    - Parse.ly and the OneTrust consent script (`OtAutoBlock.js`) also sit in the critical path (PSI estimates 740 ms).
  - **Solution**:
    - Inline the critical CSS and load the rest non-blocking.
    - Split and defer `script.js`; load Parse.ly and the consent script without blocking render.

- The main article image shows up late.
  - **Baseline**: LCP (38.6 s mobile lab / 2.9 s field — the field assessment fails on LCP).
  - **Cause**:
    - The lead promo image (`div.PagePromo-media img`, `dims.apnews.com`, 725×485) has no `fetchpriority`, so it competes with ~2,300 other requests instead of loading first.
    - It also sits behind the render-blocking chain above.
  - **Solution**:
    - Add `fetchpriority="high"` and a preload to the lead promo image.

- The page can't be used while it's still loading.
  - **Baseline**: TBT (1,250 ms mobile / 4,650 ms desktop lab); main-thread work 21.2 s, JS execution 5.9 s.
  - **Cause**:
    - Third-party ad and tracking scripts saturate the main thread — Rubicon 776 ms, Google Tag Manager 575 ms, Web Content Assessor 524 ms, Quantcast 491 ms, pub.network 423 ms.
  - **Solution**:
    - Load ad and analytics scripts after first paint, lazy-load below-fold ad slots, and break up the long tasks.

- Images don't load in the order the reader needs.
  - **Baseline**: Performance / LCP.
  - **Cause**:
    - Only 2 of 106 images set a priority, so the browser can't tell the lead story image from ad creatives and below-fold photos — ads and secondary images often download before the headline image.
  - **Solution**:
    - `fetchpriority="high"` on the lead image and `fetchpriority="low"` / `loading="lazy"` on the rest.

- The consent wall makes the page look broken on first load.
  - **Baseline**: LCP / perceived load.
  - **Cause**:
    - The OneTrust consent modal ("603 partners") covers the article on the first mobile load; its script is render-blocking (900 ms) and the ad stack waits on the consent choice, so the top of the page sits empty until it resolves.
  - **Solution**:
    - Paint the article underneath the modal immediately, load the consent script without blocking render, and hold non-essential vendors until after the user chooses.

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

- Repeat visits barely get faster.
  - **Baseline**: repeat-view transfer 7.2 MB (vs 34.3 MB fresh).
  - **Cause**:
    - First-party assets cache well, but the 2,000+ third-party ad/tracker requests are uncacheable and re-download on every visit.
  - **Solution**:
    - Caching can't fix this — reduce the ad/tracker calls themselves (fewer vendors, lazy-load, consolidate).

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

- Tap targets are too small and too close together.
  - **Baseline**: the Lighthouse accessibility audit flags 49 undersized or crowded touch targets. Mobile-only: a mouse pointer doesn't care, a thumb does.
  - **Cause**:
    - Nav links, share buttons and article-card links are sized and spaced for a cursor, so on a phone they fall under the ~24 px minimum and are easy to mis-tap.
  - **Solution**:
    - Give tap targets at least 24×24 CSS px (48×48 recommended) with enough spacing at the mobile breakpoint.

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
