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

### Source A: NWS Point Forecast
`https://forecast.weather.gov/MapClick.php?lat={lat}&lon={lon}`

Provides: 7-day text forecast, wind direction/speed, high/low temps,
current conditions from nearest station, hazard alerts.

### Source B: NWS Hourly XML Forecast (dew point, days 1-2)
`https://forecast.weather.gov/MapClick.php?lat={lat}&lon={lon}&FcstType=digitalDWML`

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

### Source C: WeatherBug Hourly Forecast (dew point, days 1-7 — unreliable via fetch, verify each use)
For **Lavallette NJ**: `https://www.weatherbug.com/weather-forecast/hourly/lavallette-nj-08753?day={today|tomorrow|3|4|5|6|7}`
For **Katonah NY**: `https://www.weatherbug.com/weather-forecast/hourly/katonah-ny-10536?day={...}`
For **other locations**: construct as `https://www.weatherbug.com/weather-forecast/hourly/{city}-{state}-{zip}?day={...}`

**Caching caveat (as of 2026-08-15):** the `day=3/4/5/6/7` params work
correctly when checked directly in a browser, but repeated fetches through
the fetch tool returned content identical to `day=tomorrow` regardless of
which day param was requested — the same stale-cache behavior observed on
the NWS point-forecast page in this same session. Root cause unconfirmed
(fetch-tool-side cache vs. something else), but it's NOT a broken/dead
URL — don't avoid these params outright. Instead: after fetching, sanity
check the returned date/day-label in the content against what was
requested before trusting the dew point numbers. If they don't match,
treat the fetch as stale and don't rely on it for that day.

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
   `https://forecast.weather.gov/product.php?site=NWS&issuedby={CODE}&product=AFD&format=CI&version=1&glossary=1&highlight=off`

Common codes for reference: `phi` (Philadelphia/NJ Shore), `okx` (NY/Westchester),
`fgz` (Flagstaff/Sedona AZ), `lot` (Chicago), `bos` (Boston) — but always
derive from the point forecast page rather than guessing.

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
- NWS AFD explains *why* and *when* the pattern changes — use it for context
- Dew point is ground truth; wind direction is the leading indicator of trend
- Best beach days: post-frontal NW, dew point <63°F, deep blue sky
- Worst beach days: SW wind, dew point 70°F+, overnight low near 75°F+
- The overnight low progression across the week is the clearest signal of
  whether the air mass is drying out or building moisture
- Always close the report with a Sources list and each source's
  update/fetch time (Step 5)
