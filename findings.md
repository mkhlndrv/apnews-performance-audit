# Findings

Findings from the [baseline](baseline.md), grouped by area, worst first within
each. Rendering numbers come from the PageSpeed Insights mobile Insights and
Diagnostics; networking numbers come from the DevTools Network panel (both for
https://apnews.com/).

## Page load & rendering

- The main article image appears far too late.
  - **Metric**: LCP (38.6 s lab, 2.9 s field — the field assessment fails on LCP).
  - **Cause**: PSI's LCP element is the lead article promo image
    (`div.PagePromo-media img`, `dims.apnews.com`, 725×485). It is discoverable in
    the HTML and not lazy-loaded, but `fetchpriority="high"` is not applied, so it
    isn't fetched ahead of the ad and tracking traffic.
  - **Solution**: add `fetchpriority="high"` (and a preload) to the lead promo
    image, and `fetchpriority="low"` / `loading="lazy"` to the images below it.

- The screen stays blank for several seconds before anything renders.
  - **Metric**: FCP (6.3 s). PSI "render-blocking requests" estimates 740 ms.
  - **Cause**: a 105.6 KiB stylesheet (`All.min.css`, 4,800 ms) and a 41.5 KiB
    first-party script (`script.js`, 2,850 ms) block the first paint, along with
    Parse.ly (1,310 ms).
  - **Solution**: inline the critical CSS and load the rest non-blocking, and
    split and defer `script.js` so the first paint doesn't wait on it.

- The page freezes and can't respond while scripts run.
  - **Metric**: TBT (1,250 ms). PSI puts main-thread work at 21.2 s and
    JavaScript execution at 5.9 s.
  - **Cause**: the main thread is saturated by first-party script plus ad and
    analytics vendors each burning hundreds of ms — Rubicon 776 ms, Google Tag
    Manager 575 ms, Web Content Assessor 524 ms, Quantcast 491 ms, pub.network
    423 ms.
  - **Solution**: load ad and analytics scripts after first paint, lazy-load the
    below-fold ad slots, and break up the long tasks.

- The page pulls in far too many ad and analytics vendors.
  - **Metric**: Speed Index (17.1 s); it also feeds TBT.
  - **Cause**: the heaviest third parties by transfer are Rubicon Project
    (385 KiB) and Google Tag Manager (330 KiB), on top of ~20 more vendors in the
    diagnostics (Viafoura, Kameleoon, Amazon Ads, Doubleclick, Wunderkind,
    Quantcast, permutive, pub.network, and others).
  - **Solution**: cut the number of ad partners, lazy-load ad slots, and defer
    non-critical vendor scripts until after the content is up.

- The consent manager blocks rendering and gates the vendor load.
  - **Metric**: FCP / perceived load.
  - **Cause**: the OneTrust/Optanon consent script (`cdn.cookielaw.org`,
    "603 partners") is render-blocking — `OtAutoBlock.js` alone costs 900 ms on
    the critical path — and its full-screen modal covers the article on first
    load, with much of the ad stack waiting on the consent choice.
  - **Solution**: load the consent script without blocking render, paint the
    article underneath the modal immediately, and hold non-essential vendors until
    after the user chooses.

## Networking

Numbers from the DevTools Network panel (desktop, fresh load then soft refresh) —
see `baseline.md` → Network Activity.

### Good

- Text compression is working.
  - **Metric**: network compression.
  - JS and CSS ship br/gzip: 37.0 MB uncompressed comes down as 7.8 MB (~79 %
    reduction). Images aren't gzipped, which is correct — they're already
    compressed formats.

- First-party caching is working.
  - **Metric**: network caching.
  - Static assets are content-hashed with a long TTL, so a soft refresh drops
    transfer from 34.3 MB to 7.2 MB — about 79 % less over the wire.

### Worst

- One visit downloads ~34 MB across 2,338 requests.
  - **Users**: on a phone plan or a slow connection this is expensive and slow,
    and the page keeps requesting for over two minutes.
  - **Metric**: total transfer (34.3 MB) and request count (2,338); feeds Speed
    Index and TBT.
  - **Cause**: most of those requests are third-party ad, bidding and tracking
    calls (Rubicon/prebid, GTM, dozens of vendors), not content.
  - **Solution**: cut the number of ad partners and bidders, lazy-load below-fold
    ad slots, and drop non-essential trackers.

- Images are the single biggest download.
  - **Users**: images are 15.7 MB of the 34.3 MB — the bulk of what a visitor
    pulls, and slow to fill in on mobile.
  - **Metric**: image transfer (15.7 MB); LCP and Speed Index.
  - **Cause**: 714 image requests, not all in next-gen formats or sized to their
    display box.
  - **Solution**: serve WebP/AVIF, use responsive `srcset`/`sizes`, and lazy-load
    offscreen images.

- The JavaScript payload is enormous.
  - **Users**: 663 script files and 37 MB of uncompressed JS to download, parse
    and run — this is what makes the page freeze (see the TBT finding above).
  - **Metric**: JS transfer 7.8 MB / 37.0 MB uncompressed, 663 requests; TBT.
  - **Cause**: third-party ad and tracking scripts dominate both the count and
    the volume.
  - **Solution**: defer and cut third-party JS, load it after first paint, and
    remove unused code.

- Caching can't touch the part that hurts.
  - **Users**: a repeat visit still pulls 7.2 MB, because the ad and tracker
    calls re-run every time even though the site's own assets come from cache.
  - **Metric**: repeat-view transfer (7.2 MB vs 34.3 MB fresh).
  - **Cause**: the 2,000+ third-party ad/tracker requests are uncacheable by
    design.
  - **Solution**: the lever isn't caching — it's making fewer ad/tracker calls
    (fewer vendors, lazy-load, consolidate).
