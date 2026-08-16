# Is the Earth really shaking more?

A single-page, zero-dependency tracker for global earthquake frequency, built to answer one
question honestly: **is seismic activity actually rising, or is it just getting more coverage?**

Everything is fetched live in the visitor's browser from the
[USGS FDSN event API](https://earthquake.usgs.gov/fdsnws/event/1/) — the public interface to the
ANSS Comprehensive Catalog. Nothing is precomputed, nothing is stored server-side, there is no
build step and no backend.

## Deploying to Netlify

Drag this folder onto Netlify, or point a site at it. `netlify.toml` sets `publish = "."` with no
build command, plus security headers and a CSP.

The CSP was tested against the real page with the exact production headers: zero violations. If you
ever add an external script, image host, or API, update `connect-src` / `script-src` accordingly or
the page will silently break.

## How the data loads

| Slice | API query | Measured size (gzipped) |
|---|---|---|
| Default (Moderate + Strong + Major) | `minmagnitude=5` | ~0.25 MB per year |
| Opt-in "Light" band | `minmagnitude=4.5&maxmagnitude=4.99` | ~0.63 MB per year |

One request per year, three concurrent, newest year first, with exponential-backoff retry on
429/503. Charts render progressively as each year lands rather than after one long wait. Completed
years are cached in IndexedDB, so repeat visits are instant; if IndexedDB is unavailable (private
mode, older browser) every cache call degrades to a no-op and the page still works.

The Light band is off by default because it roughly quadruples the download for data that doesn't
change the answer. The chip shows the real cost for the currently selected period.

Year-chunking also keeps every response well under the API's 20,000-event ceiling.

## Design decisions worth knowing

**The magnitude ramp is validated, not eyeballed.** The four classes use a single-hue ordinal blue
ramp checked against this page's exact dark surface (`#0e1014`) with the `dataviz` skill's
validator: lightness monotone, adjacent ΔL ≥ 0.06, light end 2.87:1 contrast, hue spread 3°. All
pass. Brighter = bigger, so rare large events pop off a recessive field of small ones.

**PAGER alert colours are the fixed status palette and never carry meaning alone.** Green and red
sit ΔE 4.1 apart under deuteranopia, so every alert tag ships a glyph *and* a text label.

**Magnitude classes use the real USGS names** — Light, Moderate, Strong, Major — rather than
invented low/medium/high, because those are the terms the source catalog and seismologists use.

**Each class also gets its own panel with its own scale.** Small quakes outnumber large ones about
ten to one per magnitude step, so on a shared axis M7+ is a hairline. Small multiples are the only
honest way to show that no individual class is trending.

**Non-earthquake events are excluded.** The catalog includes quarry blasts, explosions and nuclear
tests; the page filters on `type === "earthquake"`. This is why counts here can differ by one or two
from a raw API count — e.g. the 2017 North Korea test registers around M6.3 and is correctly dropped.

## The three honesty guards

These exist because each one was a bug found during testing, and each would have produced a
confidently wrong headline:

1. **The current year is compared like-for-like.** By default the page compares 1 Jan → today
   against exactly that window in every earlier year. Switching to whole calendar years would make
   the unfinished year look collapsed, so in that mode the verdict judges the **last complete year**
   instead and says so.
2. **The "same window" filter never reaches the monthly chart.** Applying it there would zero out
   every September-to-December and invent a crash that isn't in the data.
3. **Thin selections refuse to return a verdict.** Filter down to orange/red alerts and you get
   ~4 events a year; a z-score on that is noise, so the page says "too few events to call" rather
   than dressing it up.

The verdict band itself is measured, not hand-picked: it is the z-score against the mean and
standard deviation of the completed years currently in view.

## Known limitation

There is no live byte counter. USGS sends no `Timing-Allow-Origin` header, so
`PerformanceResourceTiming.transferSize` is always `0` cross-origin, and counting decoded JSON
length would overstate the real gzipped transfer roughly sixfold. The per-year estimates in the
table above are measured from `curl` and are what the opt-in chip displays.

## Verification

Numbers were cross-checked against independent `count` API queries:

| Figure | Page | API |
|---|---|---|
| M7.0+ 2016 → Aug 2026 | 145 | 145 |
| M6.0–6.9 same span | 1,278 | 1,279 (−1 = NK nuclear test, correctly excluded) |
| M5.0+ 2026 Jan 1–Aug 16 | 1,241 | 1,241 |
| M4.5+ 2026 full year to date | 4,645 | 4,645 |
