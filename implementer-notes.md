# Implementer notes — apnews.com

This is the technical layer of the audit, written for the developer who picks
up an approved finding and has to make it true. It answers one question per
finding: what do I change, and how will I know it worked. The business case
lives in [stakeholder-report.md](stakeholder-report.md); the findings catalog
with evidence and scores is [findings.md](findings.md). All three documents
share finding IDs F-01–F-18, and the same ID carries the same numbers
everywhere.

Two constraints to know up front. This is a black-box audit: names below are
the shipped asset names (`All.min.<hash>.gz.css`), selectors and hosts as
observed in production — mapping them to source files happens on your side of
the repo wall. And the audited pages are the homepage (`/`) plus one article
page for the rendering checks; other templates weren't measured.

## The bench — reproduce what we saw

Every number in the findings was captured under one of these setups. Before
starting a finding, reproduce its number once so you have your own baseline;
after the change, run the same bench, not a different one.

- **Field (CrUX)** — pagespeed.web.dev for `apnews.com`, the "real users"
  panel. 75th percentile, ~28-day window, updates monthly. This is the
  slowest feedback loop (4–6 weeks to confirm) but the one that counts.
  Reference: mobile LCP 2.9 s / desktop 3.6 s, INP 155–178 ms, CLS ≤ 0.06.
- **Mobile lab (PSI)** — pagespeed.web.dev lab section: emulated Moto G
  Power, Slow 4G, 4× CPU. Clean and reproducible because it runs on
  Google's machines. Reference: Performance 31, FCP 6.3 s, LCP 38.6 s,
  TBT 1,250 ms, SI 17.1 s.
- **Local Lighthouse** — DevTools, fresh Incognito, extensions off, applied
  (not simulated) Slow 4G + 4× CPU. **Run three times and take the median**
  — on this site single runs swing FCP 7–20 s and LCP 40–52 s, and one of
  our three runs couldn't finish scoring at all (`NO_TTI_CPU_IDLE_PERIOD`:
  the main thread never went idle). A single local run proves nothing here.
- **Device lab (WebPageTest)** — iPhone 15, Chrome, 4G (9 Mbps, 170 ms RTT).
  Use Start Render / LCP / TBT and the filmstrip; ignore Document Complete —
  our run hit WPT's 30 s page-load timeout (537 requests, 8 MB, still
  loading). Reference: Start Render 3.4 s, TBT 3.25 s.
- **DevTools inspection** — fresh Incognito. Network panel with the
  Priority column enabled; Coverage tab (fresh load, no interaction —
  numbers drift a few points run to run); Performance trace with the Frames
  track for the freeze findings; Layers panel for compositing. "Disable
  cache" on for fresh-load work, off for cache work.

**Scope flags** on each work order: **local** = a template/CSS/attribute
change you can ship alone; **structural** = touches the build pipeline, a
vendor contract, or a policy — take it to your lead first, it's a different
conversation.

## Work orders

### F-01 — Lead image loads late (LCP)

- **Mechanism**: the LCP element is the lead promo image
  (`div.PagePromo-media img`, served by `dims.apnews.com`, 725×485 slot).
  It's in the initial HTML (discoverable, not lazy-loaded) but carries no
  `fetchpriority`, so the browser schedules it as one of ~2,300 competing
  requests, behind the render-blocking chain (F-02). The cost is load
  delay, not decode.
- **Change**: add `fetchpriority="high"` to the lead promo `<img>` in the
  homepage (and article-lead) template — only the lead, or the hint stops
  meaning anything. Optionally add `<link rel="preload" as="image">` for it
  in the head; since the image is responsive use `imagesrcset`/`imagesizes`
  on the preload, and keep the URL generation identical to the `<img>` or
  the preload double-downloads.
- **Verify**: DevTools Network, Priority column: the image goes
  High/Highest and starts before ad creatives. Then PSI mobile: the LCP
  load-delay component (currently the dominant share) shrinks. Field LCP
  confirms in the next CrUX cycle.
- **Scope**: local. Effort E=1 (attribute-level).

### F-02 — Blank screen before first render (FCP)

- **Mechanism**: first paint waits on the render-blocking set:
  `All.min.<hash>.gz.css` (105.6 KiB br over ~4,800 ms on Slow 4G),
  first-party `apcdp.apnews.com/script.js` (41.5 KiB, no async/defer,
  ~2,850 ms), Parse.ly (~1,310 ms) and OneTrust `OtAutoBlock.js` (~900 ms).
  PSI estimates 740 ms of blocking savings; on the device lab nothing
  paints until ~3.4 s.
- **Change**: `defer` the `apcdp` script (verify with the data-platform
  owner that nothing reads it synchronously); load Parse.ly async. The
  OneTrust auto-block script is the hard one — it exists to run before
  vendor tags, so don't blindly defer it: move vendor tags behind the
  consent callback instead, then the auto-blocker no longer needs to block
  paint (coordinate with privacy). CSS leaves the blocking path via
  F-03/F-06.
- **Verify**: Performance trace: no render-blocking request finishes after
  FCP; WPT filmstrip Start Render moves from ~3.4 s toward ~2 s
  (lab-estimated range 1–1.5 s off); PSI mobile FCP from 6.3 s.
- **Scope**: script deferrals local; the consent-flow change is structural
  (privacy/legal in the loop). Effort E=3 (~2–3 weeks with F-03/F-06).

### F-03 — One ~770 KB site-wide stylesheet, 88% unused

- **Mechanism**: a single `All.min.<hash>.gz.css` (~770 KB raw, 105.6 KiB
  br) styles every template on the site and blocks rendering on every
  page. Coverage on the homepage: 88% unused (694 KB of 788 KB) — the
  selectors belong to article, gallery, search and donate templates that
  aren't on the page. It can only grow, because everything lands in one
  file.
- **Change**: split the sheet along template lines in the build (Brightspot
  theme), so a page ships its own CSS plus a small shared core; purge
  selectors that no live template uses. Ship it behind a diff of rendered
  pages (visual regression on homepage + article at minimum) — dead-code
  removal in CSS is only safe with screenshots to prove it.
- **Verify**: Coverage unused% on the homepage drops well under half;
  transfer size of CSS on `/` drops from 105.6 KiB; FCP follows (bench:
  PSI mobile, median local run).
- **Scope**: structural — build pipeline and design-system ownership.
  Effort E=3.

### F-04 — Barely loads under mobile conditions (program acceptance)

- **Mechanism**: no separate cause — this is F-01/02/03/05/06 meeting a 4×
  slower CPU and a slow radio. Reference numbers: PSI mobile 31/100, FCP
  6.3 s, LCP 38.6 s, SI 17.1 s; local applied-throttle medians agree (FCP
  13.0 s, LCP 42.3 s).
- **Change**: none of its own. This is the acceptance condition for the
  work above: after each fix lands, re-run the mobile bench, not the
  desktop one.
- **Verify**: PSI mobile + one local median-of-3 + a WPT filmstrip,
  compared against the reference table in [baseline.md](baseline.md).
- **Scope**: program-level. Effort E=5 spans the constituent fixes.

### F-05 — Responsive images without `sizes`

- **Mechanism**: 68 of 100 `<img>` carry `srcset` but none set `sizes`, so
  the browser assumes the image is 100 vw and picks an oversized
  candidate. The bytes are wasted before layout ever runs; card images pay
  the most.
- **Change**: measure each image slot's rendered width at the breakpoints
  (DevTools, hover the element per viewport) and write the matching
  `sizes` attribute into the promo/card templates. This is a per-template
  attribute, not a redesign.
- **Verify**: Network panel: selected `srcset` candidates drop to
  near-slot width (the `dims.apnews.com` URL carries the requested size);
  fresh-mobile image transfer (currently 8.4 MB) drops. No visual change —
  spot-check a card at each breakpoint.
- **Scope**: local. Effort E=2.

### F-06 — No critical CSS; the full sheet gates paint

- **Mechanism**: the `<head>` holds only trivial inline styles (a Viafoura
  font-size override, `--button-border-radius`) and a font preload — no
  above-the-fold CSS — so first paint waits for the entire F-03 sheet.
- **Change**: extract the above-the-fold rules for the homepage and
  article templates, inline them in the head, and load the full sheet
  non-blocking (`rel="preload"` + swap, or split delivery once F-03
  lands). Generate the critical set from the rendered page, not by hand,
  and regenerate it in the build so it can't rot.
- **Verify**: Performance trace: FCP fires before the main stylesheet
  finishes downloading; PSI mobile FCP/SI move together with F-02.
- **Scope**: structural with F-03 (build integration). Effort E=3.

### F-07 — Multi-second freezes during load

- **Mechanism**: the Frames track shows a 1,733 ms frame and a 5,058 ms
  frame during load; across the ~49 s throttled load the main thread runs
  14.5 s of scripting. Attribution: `[unattributed]`/ad code ~23 s,
  ftstatic 2.2 s, ad-score 0.8 s, DoubleVerify 0.4 s — first-party is only
  ~0.3 s. One vendor (LongTail/JW video ads) ships 46 MB of video into the
  page during load.
- **Change**: nothing first-party to optimize — the lever is loading
  policy: initialize video players and ad slots on visibility
  (IntersectionObserver), after first paint; cap simultaneous players. The
  long tasks are inside vendor bundles you can't split — you can only
  decide when and whether they run.
- **Verify**: re-run the same Performance trace (fresh Incognito, applied
  throttle): the multi-second frames disappear from the Frames track;
  scripting time during the first 10 s drops sharply.
- **Scope**: structural — loading policy for revenue vendors; the vendor
  list itself is F-16's program. Effort E=3.

### F-08 — Tap targets under size on mobile

- **Mechanism**: the Lighthouse accessibility audit counts 49 touch
  targets under ~24 px or too close — nav links, share buttons,
  article-card links — sized for a cursor at the mobile breakpoint.
- **Change**: at the mobile breakpoint give interactive elements min
  24×24 CSS px (48×48 where the design allows) and spacing; mostly padding
  on existing selectors.
- **Verify**: re-run the Lighthouse accessibility audit: the tap-target
  check passes, count 49 → 0.
- **Scope**: local. Effort E=2.

### F-09 — Images load in the wrong order

- **Mechanism**: only 2 of 106 images declare any priority; 56 are
  `loading="lazy"` but the rest — ad creatives, below-fold photos —
  compete with the lead image at default priority.
- **Change**: extend `loading="lazy"` to everything below the fold and add
  `fetchpriority="low"` to ad-slot creatives; pairs with F-01's `high` on
  the lead. One template pass.
- **Verify**: Network waterfall on a fresh throttled load: the lead image
  is among the first image requests; below-fold photos start after
  viewport approach.
- **Scope**: local. Effort E=2.

### F-10 — Main thread saturated; page can't respond during load (TBT)

- **Mechanism**: lab TBT 1,250 ms mobile / 4,650 ms desktop; total
  main-thread work 21.2 s, JS execution 5.9 s. The blocking time is inside
  vendor scripts — Rubicon 776 ms, GTM 575 ms, Web Content Assessor
  524 ms, Quantcast 491 ms, pub.network 423 ms — across ~20+ more. The
  long tasks are theirs; you control scheduling, not their internals.
- **Change**: gate the ad/analytics stack behind first paint and consent
  (single loader with an explicit order), lazy-init below-fold slots
  (with F-07), and drop vendors that load but aren't used (Coverage lists
  ntv.io 729 KB, reCAPTCHA 598 KB, Primis HLS 496 KB, sekindo 482 KB
  unused at load). Several vendors load synchronously today (Primis,
  Dianomi, a Parse.ly experiments script, Riverdrop) — make them async as
  the first, cheapest step.
- **Verify**: PSI mobile TBT (median of the lab's own runs) — direction:
  under ~600 ms after policy changes, lab-estimated; the desktop 4,650 ms
  should fall proportionally. Re-trace: longest task during load shrinks.
- **Scope**: structural — shared program with F-07/F-16. Effort E=4.

### F-11 — Controls and images invisible to assistive tech

- **Mechanism**: 2 buttons and 2 links with no accessible name, 4 images
  without `alt`, 1 iframe without `title`, 29 `aria-hidden` regions
  containing focusable descendants, one heading-order break, one contrast
  failure. Element list: re-run the Lighthouse accessibility audit and
  expand each failing check — it names every node.
- **Change**: `aria-label` on the icon-only controls, `alt` text (empty
  `alt=""` only where decorative), `title` on the iframe, `tabindex="-1"`
  or removal for focusables inside `aria-hidden`, fix the heading level,
  darken the failing color token.
- **Verify**: Lighthouse accessibility re-run: the six name/alt/title
  checks pass and the score (75 mobile) moves accordingly; tab through the
  header with a screen reader for the aria-hidden fix.
- **Scope**: local. Effort E=2.

### F-12 — Interactivity attaches long after paint

- **Mechanism**: the page is fully server-rendered with no hydration
  framework, yet behaves like a hydration-heavy app: `All.min.js` plus the
  vendor layer executes for seconds after paint, and widgets (search,
  menus, embeds) only become live once it finishes. Field INP 155–178 ms
  sits borderline; content is visible well before it responds.
- **Change**: attach interactivity per component instead of per page —
  initialize widgets on first interaction or visibility (event delegation
  up front, real handlers on demand). This is an incremental islands
  pattern on top of markup that already server-renders — no framework
  migration involved.
- **Verify**: manual under throttle: click search/menu during load — it
  responds; lab TBT share of `All.min.js` drops; field INP in the next
  CrUX window.
- **Scope**: structural (front-end architecture decision), incremental to
  ship. Effort E=4.

### F-13 — Consent wall blanks the first visit

- **Mechanism**: the OneTrust modal ("603 partners") covers the page on
  first load while `OtAutoBlock.js` blocks render ~900 ms and the ad stack
  waits on the choice — so a first-time reader gets blank, then a wall,
  then the page.
- **Change**: paint the article under the modal (the content doesn't need
  consent), load the consent UI non-blocking with reserved space, and keep
  vendor tags behind the callback (with F-02). Shrinking the 603-partner
  list is a legal/ad-ops decision, not a code change — raise it, don't own
  it.
- **Verify**: device-lab filmstrip on a cleared profile: article text
  visible at/before the modal, not after; `OtAutoBlock.js` gone from the
  render-blocking list in the trace.
- **Scope**: structural (privacy/legal + ad ops). Effort E=3.

### F-14 — 20 composite layers holding 814 MB of GPU memory

- **Mechanism**: the Layers panel shows 20 layers / 814 MB; the document
  rootScroller alone is 383 MB and carries page-wide slow-scroll regions —
  non-passive touch/wheel listeners, so every scroll waits on JS. Other
  layers: a 1096×618 video, a 600×500 iframe, the sticky header
  (`.Page-header-stickyWrap`), three Bounce Exchange scroll-shades, the
  usablenet widget (`usntA40…`). Promotions are legitimate; the cost is
  volume plus the listeners.
- **Change**: register first-party wheel/touch listeners with
  `{ passive: true }` (find owners via DevTools → the scroll-blocking
  warning names the script); for vendor listeners (Bounce Exchange) it's a
  config/vendor ask. Unmount hidden ad/video iframes instead of stacking
  them.
- **Verify**: Layers panel: count and memory down (direction, not a
  promised number); Performance trace during scroll: no "handler took N
  ms" scroll warnings; the slow-scroll shading on the rootScroller gone.
- **Scope**: first-party listeners local; vendor listeners belong to the
  F-16 conversation. Effort E=2.

### F-15 — Empty client-rendered placeholders

- **Mechanism**: `#riverdrop-widget` ships as
  `<div style="min-height:600px"></div>` and is filled by a third-party
  embed (`client.riverdrop.com`); `#desktop-entitlements` (Zephr paywall
  state) is also client-filled. Until those scripts run — late, they're
  deferred behind everything — the page shows reserved blank slots; with
  the embed blocked, a 600 px hole.
- **Change**: server-render a static fallback into each slot (recirculation
  links; signed-out entitlement state) and let the embed upgrade it in
  place; collapse the `min-height` when the embed fails. The reserved
  height itself is correct (it's why CLS passes) — keep the reservation,
  fill it.
- **Verify**: reload with JS disabled: content or fallback in the slot, no
  600 px blank; on Slow 4G the slot never renders empty-then-filled.
- **Scope**: local per slot (template + one vendor integration each).
  Effort E=2.

### F-16 — One visit downloads 34 MB across 2,338 requests

- **Mechanism**: the fresh desktop load transfers 34.3 MB over 2,338
  requests (mobile: 20.4 MB / 3,169); ~46 third-party domains and 800+
  script requests. Biggest single blocks: images 15.7 MB, vendor JS with
  four bundles ≥ ~480 KB unused at load (ntv.io, reCAPTCHA, Primis,
  sekindo), one video vendor shipping 46 MB (F-07). Only ~38% of
  downloaded JS/CSS executes.
- **Change**: this is the structural program — an inventory-driven vendor
  decision (who stays), then loading policy for who remains (async, post-
  paint, on-visibility). Engineering prepares the inventory (the PSI
  third-party table plus the request log in
  [baseline.md](baseline.md)) and implements policy; the cut list is a
  product/revenue decision. Phase 1 (policy only, no contracts touched) is
  implementable now: F-07 + F-10 changes.
- **Verify**: fresh-load transfer and request count after each phase;
  policy phase alone should take single-digit MB and hundreds of requests
  off (lab-estimated); the vendor-cut phase scales with the list.
- **Scope**: structural — the program. Effort E=5.

### F-17 — First-party JS is one un-split bundle

- **Mechanism**: `All.min.<hash>.gz.js` (~450 KB raw, pre-gzipped,
  content-hashed) is 84% unused on the homepage (381 KB of 452 KB) — one
  concatenated file for every template, plus the blocking `apcdp` script
  (F-02) and a deferred `webcomponents-loader`. Tree-shaking can't reach
  inside a deliberate concatenation.
- **Change**: split per route/template in the build and lazy-load
  interaction-only components (with F-12). Honest framing: first-party JS
  is ~0.3 s of the main-thread cost — this is hygiene and an enabler for
  F-12, not the TBT fix; don't sell it as one.
- **Verify**: Coverage: unused share of first-party JS drops; per-page JS
  transfer drops from ~130 KB br; no regression in TBT (it shouldn't move
  much — see above).
- **Scope**: structural (build pipeline, with F-03). Effort E=4.

### F-18 — Repeat visits barely benefit from caching

- **Mechanism**: first-party caching already works (content-hashed, long
  TTL — refresh drops to 7.2 MB). The residual is 2,000+ third-party
  ad/tracker requests that are uncacheable by design (`no-store`, short
  TTLs, cache-busting params) and re-download every visit.
- **Change**: nothing to configure on AP's side makes vendor calls
  cacheable, and a service worker must not cache them. The only fix is
  fewer calls — this resolves through F-16. Treat this entry as the
  explanation your lead will ask for when someone suggests "just cache
  it."
- **Verify**: refresh-load transfer after F-16 phases (today 7.2 MB).
- **Scope**: structural via F-16. Effort E=4 as scored, but it has no
  standalone work.

## Domains reviewed with no work orders

Every domain the audit covered, where it landed, and — where there's no
corrective — why that's a considered conclusion, not a gap:

| Domain | Status |
|--------|--------|
| Rendering pipeline | Corrective: F-02, F-06, F-07, F-14 |
| Metrics & measurement | The bench above; conditions in [baseline.md](baseline.md) |
| Compression | **No finding — verified good.** br/gzip on all text, ~79% wire savings; images correctly not gzipped |
| HTTP protocol | **No finding — verified good.** h2 + h3 (Alt-Svc) on the main origins; a few stragglers on h1.1 aren't load-bearing |
| First-party caching | **No finding — verified good.** Content-hashed, long-TTL; refresh 34.3 → 7.2 MB. Third-party residual is F-18 |
| Service workers / offline | **Considered; does not apply.** No service worker is registered on the audited pages (none observed in any capture), and we don't recommend adding one: the HTML is already edge-cached (120 s / 1 yr per route), first-party assets already cache long, and the weight that hurts (F-16) is third-party and must not be SW-cached. Offline reading of a live news feed is a product feature, not a perf fix |
| Fonts | **Considered; no corrective.** Custom font is preloaded (`APVarW05`), no flash-of-invisible-text visible in the filmstrip; two Google Fonts stylesheets ride the third-party inventory (F-16) but aren't a standalone problem |
| Bundles: JS / CSS / source maps | Corrective: F-03, F-17. Source maps: **verified correctly absent** in production (`All.min.js.map` 404s, no `sourceMappingURL`) |
| Images | Corrective: F-01, F-05, F-09. Formats verified good (WebP via the `dims` proxy) |
| Mobile | Corrective: F-04 (acceptance), F-08; bench is mobile-first throughout |
| Accessibility | Corrective: F-08, F-11 |
| Animations | **Considered; no corrective.** The two load-time animations run on composite-friendly properties; jank comes from scripting (F-07), not animation |
| Compositing / layers | Corrective: F-14 |
| Progressive enhancement | Mixed: the site works with JS disabled (verified — strength), except the F-15 placeholder slots |
| State management & data fetching | **Considered; does not apply as a first-party concern.** The audited templates are server-rendered documents with no client data layer of their own; client-side fetching on these pages is the ad/consent/embed stack, covered by F-10/F-13/F-16 |
| Rendering strategy | **Verified good** per page type (SSR + per-route edge caching; no hydration tax) — see [baseline.md](baseline.md). Residuals: F-12, F-15 |

## Effort basis

The E in each RICE score is calendar-honest implementer effort including
testing: **E=1** an attribute-level change (F-01); **E=2** a contained
template/CSS pass (F-05, F-08, F-09, F-11, F-14, F-15); **E=3** a
build/loading-path rework shippable in ~2–3 weeks (F-02, F-03, F-06, F-07,
F-13); **E=4** a restructuring with dependencies (F-10, F-12, F-17, F-18);
**E=5** an org-level program (F-04, F-16). If your estimate after reading a
work order differs materially, say so — Effort is an input to the ranking,
and changing it in writing is how a finding legitimately moves.
