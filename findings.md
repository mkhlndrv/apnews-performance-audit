# Findings

Five findings from the [baseline](baseline.md), worst first. Numbers are from the
PageSpeed Insights mobile Insights and Diagnostics for https://apnews.com/
(screenshots in `screenshots/`).

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
