# Findings

Grouped by area, corrective findings first within each. Each finding is a
distinct, independently observable problem — where several causes feed the same
symptom, they sit under one finding. Rendering numbers come from the PSI mobile
Insights/Diagnostics, networking from the DevTools Network panel, and
accessibility from a Lighthouse accessibility audit. See [baseline](baseline.md).

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

## Accessibility

- Interactive controls and images are invisible to assistive technology.
  - **Baseline**: PSI Accessibility 75 (mobile) / 79 (desktop); a Lighthouse accessibility audit flags eight issue types.
  - **Cause**:
    - 2 buttons and 2 links have no accessible name; 4 images have no `alt`; 1 iframe has no title.
    - 29 `aria-hidden` elements contain focusable descendants; headings aren't in sequential order; 49 touch targets are too small or too close; one text/background pair fails contrast.
  - **Solution**:
    - Add `aria-label` to icon-only buttons and links, `alt` to images, and a `title` to the iframe.
    - Remove focusable children from `aria-hidden` regions, fix the heading order, enlarge/space the touch targets, and fix the contrast pair.
