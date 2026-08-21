---
name: weather-forecast
description: >
  Pull a live, personalized weather forecast with special emphasis on perceived
  comfort and humidity. Use this skill whenever the user asks about the weather,
  forecast, how humid it is, whether it's a good beach day, or any question
  about current or upcoming conditions — at their current location OR any
  location they mention. Always fetch live data; never answer from memory.
  Trigger on casual phrasing: "how's the weather", "nice out?", "beach weather
  this week?", "is it going to be sticky?", "good day to be outside?",
  "what's it like in [place]?", "will it be humid this weekend?".
---

# Weather Forecast Skill

## Overview

This skill combines NWS (structure, wind, pattern context, AND near-term
hourly dew point for days 1-2) with WeatherBug's hourly forecast (dew
point for days 1-7) to deliver a comfort-first weather interpretation
with no coverage gap across the default week-long window.

Sudeep's known locations:
- **Lavallette NJ** (beach house) — barrier island, Jersey Shore
- **Katonah NY** (home) — inland Westchester

---

## Step 1: Establish Current Date

Before anything else, call `user_time_v0` to get today's actual date and
timezone. Use this to anchor every forecast date/day-of-week reference
that follows (e.g. "today" = the date returned, "this weekend" = the
Sat/Sun following it). Never assume the date from memory or from a
prior turn in the conversation — always confirm it fresh each time this
skill runs.

## Step 2: Determine Location and Time Frame

### Location
- **If the user names a specific location**: use that, note it explicitly.
- **Otherwise**: call `user_location_v0` with `accuracy: precise`, use silently.

### Time Frame
- **Default**: today + 7-day outlook
- **If the user specifies a window** ("just today", "this weekend", "next
  Tuesday"): honor that scope. Note if it extends beyond reliable 7-day range.

---

## Step 3: Fetch Data from Multiple Sources

Fetch in parallel where possible — each source serves a different purpose.
For dew point specifically: NWS hourly XML (Source B) for days 1-2, WeatherBug
hourly (Source C) for days 1-7 — the two overlap early on and together
cover the full default window with no gap.

**General caching note:** every `forecast.weather.gov` product endpoint used
in this skill (point forecast, hourly XML, and the AFD text product below)
has been observed to occasionally serve stale, cached content — sometimes
by days, once by over a month — for a URL that had been fetched before
(by this skill or, seemingly, by anyone). Appending a throwaway
cache-busting query parameter (e.g. `&cb={random or timestamp}`) to any of
these URLs has reliably forced a fresh fetch in testing. Do this by default
on every `forecast.weather.gov` call in this skill, not just after a
staleness check fails — it's cheap insurance and the alternative is
silently reporting week(s)-old data as current.

### Source A: NWS Point Forecast
`https://forecast.weather.gov/MapClick.php?lat={lat}&lon={lon}&cb={cachebust}`

Provides: 7-day text forecast, wind direction/speed, high/low temps,
current conditions from nearest station, hazard alerts.

Note: a lat/lon that sits right on the water can resolve to a **marine**
point forecast (offshore buoy-style forecast, no land temps/comfort info)
instead of the land forecast for the town. If the returned page reads as
marine (seas in feet, boat-oriented wind categories, "NNE Atlantic City"
style zone name instead of a town name), don't treat it as a fetch
failure — nudge the coordinates or, more reliably, use the zip/city
lookup instead: `https://forecast.weather.gov/zipcity.php?inputstring={City},{ST}`.
Confirm the page is a land point forecast before pulling numbers from it.

### Source B: NWS Hourly XML Forecast (dew point, days 1-2)
`https://forecast.weather.gov/MapClick.php?lat={lat}&lon={lon}&FcstType=digitalDWML&cb={cachebust}`

Provides: hourly Dewpoint (°F), temp, RH%, wind, sky cover, precip chance —
for a ~7-day window from NWS's raw forecast grid, in XML. This is the
**primary dew point source for today and tomorrow**: it's more precise
(hourly, not just per day/night period) and comes straight from NWS
rather than a repackaged third-party feed. Use this first for near-term
dew point.

**Use the XML endpoint (`FcstType=digitalDWML`), not the HTML tabular
page (`FcstType=digital`).** The HTML tabular page has repeatedly returned
stale, cached content through the fetch tool — sometimes weeks old, and
the stale snapshot changes if the lat/lon is nudged, suggesting a
pre-rendered cache rather than a live page. The XML endpoint has
returned live, correctly dated data every time tested. Always sanity
check the `<creation-date>` in the XML response against today's date
(from Step 1) before trusting the numbers — if it doesn't match, treat
as stale and fall back per the coverage note below.

Also note: if the lat/lon resolves to a marine grid point (see Source A),
this feed can come back with wind/wave data only and no temperature or
dewpoint fields at all. That's a sign to fix the coordinates/location
(see Source A), not a sign the endpoint itself is broken.

### Source C: WeatherBug Hourly Forecast (dew point, days 1-7 — unreliable via fetch, verify each use)
For **Lavallette NJ**: `https://www.weatherbug.com/weather-forecast/hourly/lavallette-nj-08753?day={today|tomorrow|3|4|5|6|7}&cb={cachebust}`
For **Katonah NY**: `https://www.weatherbug.com/weather-forecast/hourly/katonah-ny-10536?day={...}&cb={cachebust}`
For **other locations**: construct as `https://www.weatherbug.com/weather-forecast/hourly/{city}-{state}-{zip}?day={...}&cb={cachebust}`

**Caching caveat (as of 2026-08-15):** the `day=3/4/5/6/7` params work
correctly when checked directly in a browser, but repeated fetches through
the fetch tool have returned content identical to `day=tomorrow` (or to
whatever day was fetched previously for that URL) regardless of which day
param was requested — the same stale-cache behavior observed on the NWS
pages in this skill. Appending a cache-busting query param (see general
note above) has fixed this in testing. Even with cache-busting applied,
still sanity check the returned date/day-label in the content against
what was requested before trusting the dew point numbers — if they don't
match, treat the fetch as stale and don't rely on it for that day.

Provides (when fetched fresh): dew point per hour, for each of the next 7
days. Use this — not the 10-day summary page — for dew point beyond NWS's
48-hour tabular window. The 10-day *summary* page (`.../10-day-weather/...`)
only surfaces an explicit dew point figure starting several days out
(varies — check actual content rather than assuming a fixed day), so
prefer the hourly page for anything closer in.

**Coverage**: NWS hourly XML (days 1-2, most reliable) + WeatherBug hourly
(days 1-7, verify freshness per above) together should give dew point for
the full default 7-day window. Fall back to overnight low + wind
direction/air mass reasoning only beyond day 7, on a fetch failure, or
when a WeatherBug fetch's returned date doesn't match the requested day.

### Source D: NWS AFD Discussion (when useful)
The correct NWS office varies by location — do NOT hardcode it. Instead:
1. Read the office code from the NWS point forecast page (shown as "Your
   local forecast office is [City, State]" with a link like `/phi/` or `/okx/`)
2. Extract the 3-letter office code from that link
3. Construct the AFD URL dynamically:
   `https://forecast.weather.gov/product.php?site={CODE}&issuedby={CODE}&product=AFD&format=txt&version=1&glossary=0&cb={cachebust}`

Common codes for reference: `phi` (Philadelphia/NJ Shore), `okx` (NY/Westchester),
`fgz` (Flagstaff/Sedona AZ), `lot` (Chicago), `bos` (Boston) — but always
derive from the point forecast page rather than guessing.

**Staleness — this is the source most prone to it, and the failures can be
severe (in testing, one fetch returned a discussion issued *two months*
earlier, with no indication anything was wrong).** Root cause confirmed:
the fetch tool caches by exact URL, and `product.php` calls with identical
query parameters as a previous fetch (from this skill or otherwise) can
return that old cached response indefinitely — not just within a short
TTL. This is independent of `format` (`CI` and `txt` both cache the same
way) and independent of using `site=NWS` vs `site={CODE}` (both work once
the caching is handled; prefer `site={CODE}` for consistency with the
`issuedby` param). Two things fix it, use both:
1. **Always append a cache-busting param** (`&cb={timestamp or random
   string}`, unique per fetch) — confirmed in testing to force a live
   fetch instead of a cached one, on the same URL that had just returned
   stale content without it.
2. **Always verify freshness before using the content.** Prefer
   `format=txt` — it's plain text and the issuance line is unambiguous.
   The product opens with a WMO header line like `FXUS61 KPHI 210647`
   (the 6 digits are day-of-month + issue time in UTC — `21` `0647` =
   issued the 21st at 06:47 UTC) and/or a plain-language line like
   `247 AM EDT Fri Aug 21 2026`. Compare the day against today's date
   (from Step 1). AFDs are typically re-issued every ~6 hours, so
   anything more than about a day old should be treated as suspect even
   if it's not wildly off. If the date doesn't check out, discard the
   fetch, don't cite it as a source, and fall back to the front-passage
   language already present in the Source A text forecast plus the
   wind-direction/air-mass reasoning in Step 4 — don't retry the exact
   same URL/params again; change the cache-busting value and retry once,
   otherwise proceed without it.
3. `version=0` is invalid (400 error) — `version=1` is the most recent
   product; higher numbers step further back in time.
4. `api.weather.gov` (the JSON API alternative) is blocked by that
   domain's robots.txt for the fetch tool — not a usable fallback here.

Provides: meteorologist narrative on fronts, air mass changes, pattern shifts,
relief days. Fetch when the multi-day pattern is complex or changing.

---

## Step 4: Build the Forecast Response

### Dew Point is the Ground Truth
| Dew Point | Comfort |
|---|---|
| Below 55°F | 😌 Crisp, refreshing |
| 55–60°F | 😊 Comfortable |
| 60–65°F | 🙂 Acceptable |
| 65–68°F | 😐 Noticeable — other factors decide |
| 68–70°F | 😓 Muggy |
| 70°F+ | 🥵 Oppressive — no relief regardless of wind |

### Overnight Low as Backup Proxy
When no forecasted dew point is available — beyond day 7, or a fetch
failure:
- Low in low-60s or below → dry air mass, comfortable
- Low at 70°F+ → saturated air mass, no overnight relief

### Wind Direction at the Coast (Lavallette and similar)
| Wind | Comfort Effect | Why |
|---|---|---|
| NW, N | Very good | Post-frontal Canadian air, dry |
| NE (post-frontal) | Good | Dry air wrapping around high |
| E/SE (thermal, afternoon) | Good | Local land-driven, evaporative |
| E/SE (synoptic, summer) | Deceptive — often humid | Warm Atlantic fetch |
| S, SW | Bad | Gulf moisture |
| W (post-frontal) | Neutral to good | Drier than SW |

**Key summer distinction**: A large-scale easterly in July–August is NOT
the refreshing sea breeze. Atlantic water off NJ runs 70-74°F in August —
warm enough to load passing air with moisture. The genuine refreshing
easterly is a local afternoon thermal effect, or post-frontal.

### Air Mass Origin (most important factor at the beach)
- **Post-frontal NW/N** = dry Canadian air = best days
- **SW** = Gulf moisture = worst days
- **Synoptic E/SE in summer** = warm Atlantic moisture = deceptively humid
- **Stagnant/light winds** = moisture builds

### Inland vs. Coast
- **Inland** (Katonah): dew point is the primary predictor. Simpler.
- **Coast** (Lavallette): dew point + wind direction + air mass origin
  together determine comfort. Same dew point can feel very different.

### Front Passage Signals (from AFD)
- "Post-frontal" = good news
- "Gulf moisture surging" / "warm front lifting" = bad news
- "Drier airmass by [day]" = flag as relief day
- "Dewpoints mixing out" = afternoon improvement
- Milky/hazy sky (visible in text descriptions) = humid air mass in place

---

## Step 5: Response Format

Lead with a **one-line comfort verdict** for today, then expand.

```
**[Location] — [Date]**
Today: [comfort verdict + dew point + conditions + wind]
Tonight: [low + what it signals]
This week:
  Mon: dew point X°F, [comfort label], wind [dir] — [one line]
  Tue: ...
  ...
What to watch: [front passages, pattern shifts, best/worst days]

Sources:
- NWS Point Forecast — updated [time/date from page]
- NWS Hourly XML — updated [creation-date from XML]
- WeatherBug Hourly (days 3-7) — fetched [date/time], day label verified as [day]
- NWS AFD Discussion — issued [date/time from product] (if used)
```

- If non-default location: open with "Here's the forecast for [place]..."
- If non-default time frame: note it ("Here's just the weekend...")
- Don't assume familiarity with meteorological jargon — briefly explain
  terms in plain language when first used ("post-frontal NW flow" →
  "dry air moving in behind the front", "synoptic easterly" → "a
  large-scale wind off the ocean, not the local sea breeze")
- Always flag the best day(s) of the week explicitly
- At the beach, always comment on whether wind is the real sea breeze or a
  deceptive synoptic flow
- **Always end the report with a "Sources" list** naming every source
  actually used and its update/issue/fetch time, so staleness is visible
  at a glance. Pull the timestamp from each source itself (NWS page's
  "Last Update", the XML `<creation-date>`, the product issuance time,
  or the fetch time if the source doesn't expose one) — don't guess or
  omit it. If a source came back stale and was discarded, don't list it;
  note the fallback used instead (e.g. "dew point via overnight-low
  proxy — NWS XML fetch appeared stale").

---

## Key Reminders

- Always call `user_time_v0` first (Step 1) — never assume today's date
- Dew point source hierarchy: NWS hourly XML (`FcstType=digitalDWML`) for
  days 1-2 (primary, most precise, and reliably live — NOT the HTML
  `FcstType=digital` tabular page, which has repeatedly returned stale
  cached content) → WeatherBug hourly page for days 1-7 (verify the
  returned date matches the requested day param before trusting it —
  fetches have intermittently returned stale/cached content) →
  overnight-low/wind reasoning only beyond day 7, on fetch failure, or
  when the freshness check fails
- NWS is also the structure, wind, and pattern source throughout
- NWS AFD explains *why* and *when* the pattern changes — use it for
  context. It's the least reliable fetch of the four sources (see Source
  D) — always append a cache-busting query param and verify the
  issuance date before trusting or citing it
- On every `forecast.weather.gov` URL in this skill (Sources A, B, D),
  append a throwaway cache-busting query parameter by default — don't
  wait for a staleness check to fail first
- Dew point is ground truth; wind direction is the leading indicator of trend
- Best beach days: post-frontal NW, dew point <63°F, deep blue sky
- Worst beach days: SW wind, dew point 70°F+, overnight low near 75°F+
- The overnight low progression across the week is the clearest signal of
  whether the air mass is drying out or building moisture
- Always close the report with a Sources list and each source's
  update/fetch time (Step 5)
