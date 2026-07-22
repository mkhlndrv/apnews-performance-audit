# Stakeholder report — apnews.com performance

This is the decision layer of the audit, written for the people who fund and
schedule work rather than the people who implement it. It says what the
performance problems cost the business, what fixing them buys, and in what
order. The implementer layer — mechanisms, exact fixes, effort basis — is
[findings.md](findings.md); every claim here traces to it by finding ID
(F-01…F-18) with matching numbers, and the measurement conditions are in
[baseline.md](baseline.md). Measurements: June–July 2026.

Intended use: budget justification and a ranked plan for the next one to two
quarters.

## The two-minute version

apnews.com is fast to serve and slow to show. The server answers in 0.2
seconds; the reader then waits behind stylesheets, scripts and ad calls before
the story appears. Google measures the site's real readers every month and
currently fails it on its headline speed metric — desktop and mobile alike —
while the other two user-experience grades pass.

Three numbers carry the situation:

- **2.9 seconds** — when the typical mobile reader sees the main story
  content (Google's own field data, 75th percentile). Google's bar for
  "good" is 2.0 seconds. Desktop is 3.6 s against the same bar.
- **~3.4 seconds of blank screen** — what a normal phone on 4G shows before
  anything appears. In the same device-lab run the page was still loading
  when the test gave up at 30 seconds.
- **34 MB across 2,338 requests** — one desktop homepage visit. About 46
  ad-tech and tracking vendors account for most of the requests, and the
  page runs only ~38% of the code it downloads.

The ask, in order:

| # | Purchase | What it buys | Price |
|---|----------|--------------|-------|
| 1 | Image delivery fix (F-01, F-05, F-09) | The main story visible noticeably sooner for every reader, aimed directly at the failing metric | Days; one front-end engineer |
| 2 | First-paint fix (F-02, F-03, F-06) | The blank-screen window cut down; the page starts rendering before the full machinery arrives | 2–3 weeks; front-end team |
| 3 | A decision on the third-party stack (F-07, F-10, F-13, F-16) | A page that responds while it loads, and most of the 34 MB back | Quarter-scale program; product, ad ops and legal at the table |

If this is all you read: fund 1 and 2 now — they are cheap, measured, and
aimed at the metric Google grades us on — and put 3 on an agenda, because it
is the structural cause and no amount of front-end work substitutes for that
decision.

## What is already strong

We measured before judging, and several things hold up well:

- **The publishing architecture is right.** A published article is rendered
  once and then served from CDN edge for up to a year; the homepage
  re-renders at most every 120 seconds. This is the expensive part of a news
  platform done correctly — we are *not* recommending a replatform, and that
  is money saved.
- **Repeat visits are cheap where AP controls them.** First-party assets are
  fingerprinted and cached long; a refresh transfers 7.2 MB instead of
  34.3 MB (~79% less).
- **Text compression is effective** — ~79% of JS/CSS bytes never cross the
  wire.
- **The page doesn't jump around while loading** (layout-shift grade passes
  everywhere we measured), **and once loaded it responds to input** (Google's
  responsiveness grade passes in field data).

The problems below sit on top of a sound foundation, which is exactly why
they are worth fixing: the ceiling is high.

## The five priorities

All eighteen corrective findings are scored with RICE (Reach × Impact ×
Confidence ÷ Effort; scales documented in [findings.md](findings.md)). The
top quarter by score — five of eighteen, cut mechanically, not by taste —
are the priorities. Moving an item in or out requires changing a scored
input, in writing.

| ID | Priority | Score |
|----|----------|:-----:|
| F-01 | The main story image is left to compete with 2,300 other downloads | 24.0 |
| F-02 | Nothing appears for the first seconds of every visit | 5.3 |
| F-03 | Every page ships a 770 KB stylesheet and uses about a tenth of it | 4.7 |
| F-04 | On a mid-range phone on mobile data, the site barely loads | 4.2 |
| F-05 | Photos download larger than the space they are shown in | 4.1 |

**F-01 — The main story image is last in line (readers see it at 2.9 s; the
bar is 2.0 s).** The lead image carries no loading priority, so the browser
treats it like any of the other ~2,300 downloads on the page. *Risk:* this is
the metric Google folds into ranking, assessed on mobile, on every story URL
— a persistent site-wide headwind against every competing outlet, not a
cliff. *Cost:* the fix is among the cheapest in the report — a priority
attribute and a preload; days of one engineer. *Opportunity:* the single
largest measured lever on the failing metric. Expected movement is from
2.9 s toward the 2.0 s bar; confirmed by Google's next field-data cycle
(~4–6 weeks after ship).

**F-02 — Nothing appears for the first ~3.4 seconds on a phone.** Two
first-party files and two vendor scripts must fully download before the
browser is allowed to draw anything ("render-blocking": nothing appears
until these finish). *Risk:* the blank window is where abandonment happens,
and it is silent — nobody files a complaint, they leave. Breaking-news
readers on weak connections are the peak audience and get the worst of it.
*Cost:* 2–3 weeks of front-end work, shared with F-03/F-06. *Opportunity:*
the first impression of every first-time visit stops being a white page.

**F-03 — Every page ships a 770 KB stylesheet and uses ~12% of it — and
nothing appears until it finishes downloading.** One site-wide stylesheet
carries the styling for every template on apnews.com; the homepage measures
88% of it unused. *Risk:* every page pays the full download before first
paint, forever, and the file only grows. *Cost:* splitting it is part of the
same 2–3 week package as F-02. *Opportunity:* the largest render-blocking
file mostly disappears from the critical path.

**F-04 — On a mid-range phone on mobile data, the site barely loads (31/100
on Google's standard mobile test; main image at 38.6 s).** This is not a
separate purchase: it is what packages 1 and 2 are bought *for*, and the
condition they must be verified under. Mobile is the profile Google ranks —
and the numbers say the desktop experience and the ranked experience are
different products today.

**F-05 — Photos download larger than the boxes they are shown in.** 68 of
100 images offer responsive variants but omit the one attribute that lets
the browser pick the small one, so it over-fetches. *Risk:* wasted bytes on
every metered mobile plan. *Cost:* an attribute per image template; days.
*Opportunity:* image bytes drop with zero visual change.

One finding sits just under the mechanical line: F-06, extracting the
above-the-fold CSS (score 4.0). It is the same workstream as F-03 and is
funded inside package 2 rather than as a separate ask.

## The structural problem, priced separately

**The advertising and tracking stack is the root cause of most of the above
— and it is a business decision, not an engineering ticket** (F-07, F-10,
F-13, F-16; also why F-18 can't be fixed by caching).

The evidence, stated so it can be checked: ~46 distinct third-party domains
and 800+ script requests per homepage load; one video-ad vendor alone ships
46 MB; during load the page freezes outright for 1.7 s and then 5.1 s; only
~38% of all downloaded code ever runs; and the first thing a new reader sees
is a consent wall naming 603 partners while the article waits underneath.
Tracker and bidding calls are uncacheable by design, so repeat visits
re-download them every time.

*Risk while it stands:* every front-end win above has a ceiling. Image and
CSS work cannot make the page responsive while ad and video scripts saturate
the processor — of a ~49-second throttled load, 14.5 seconds is script
execution, almost none of it AP's own code. The freezes land during reading,
and a frozen page shows ads nobody can scroll to.

*Cost:* honest answer — high. This is vendor consolidation and loading
policy: which partners stay, which load after the story, which lazy-load
below the fold. It needs product, ad ops and legal in the room, and it
touches revenue contracts. That effort is exactly why its ROI score ranks it
below the quick wins (2.2–3.7), not because it matters less. It is the
paradox of the current setup: the machinery that monetizes attention is the
main thing suppressing it.

*Opportunity:* most of the 34 MB, the multi-second freezes, and the headroom
that keeps the responsiveness grade passing as the site grows. A gradient is
available — start with loading policy (defer and lazy-load, no contracts
touched), measure, then negotiate the vendor list from data.

## What we are not asking for

No replatform and no rendering rewrite. We verified the architecture
per page type and it is the correct one; a migration would be the most
expensive item on any menu and would fix none of the measured problems. The
recommendation is to spend on the five priorities and the third-party
decision — not on rebuilding what works.

## The three objections we expect

- **"Readers aren't complaining."** Abandonment is silent; the complaint
  channel is Google's field data, and it currently reads 2.9 s against a
  2.0 s bar, refreshed monthly, public.
- **"It's fast on my machine."** Office fiber on a desktop is not what gets
  ranked. Google assesses the 75th-percentile mobile reader — the numbers
  above *are* that reader.
- **"Can't we just cache more?"** Caching is already one of the site's
  strengths (repeat visits drop to 7.2 MB) and it cannot touch this: ad and
  tracker calls are uncacheable by design and re-fetch on every visit, and
  first visits — the majority for a news site arriving from search and
  social — never benefit at all.

## Check the numbers yourself

No trust in this report is required for the headline claims:

- Type `apnews.com` into **pagespeed.web.dev**. The "what real users
  experienced" panel is Google's field data — the 2.9 s and the pass/fail
  grades above, from Google, not from us.
- Open `screenshots/06-webpagetest/` in this repo: the film strip shows the
  blank frames until ~3.4 s on an iPhone on 4G, and the run timing out at
  30 s.
- Open apnews.com on your own phone over mobile data (browser, not the app)
  and count the seconds before the lead story is readable.

## What a yes looks like

Three things turn this report into a plan: an **owner** per package, a
**sequence** (package 1 this sprint, package 2 next, the third-party agenda
scheduled), and a **date** — field data re-measured about six weeks after
packages 1–2 ship, since Google's numbers lag by roughly a month. The same
table then shows whether the 2.9 s moved, in public data neither we nor the
team can massage.

## Method note

Scores come from the RICE table in [findings.md](findings.md); the
confidence column marks the soft numbers honestly (third-party items carry
0.6–0.7 because outcomes depend on vendors and business choices, not only on
code). Lab numbers use Google's standard mobile stress test (Slow 4G, 4×
CPU slowdown); the device-lab run is WebPageTest on an iPhone 15 over 4G;
field numbers are Google CrUX at the 75th percentile. Full capture
conditions, run-to-run variance, and every screenshot: [baseline.md](baseline.md).
