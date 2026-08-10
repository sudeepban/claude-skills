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

## Step 1: Determine Location and Time Frame

### Location
- **If the user names a specific location**: use that, note it explicitly.
- **Otherwise**: call `user_location_v0` with `accuracy: precise`, use silently.

### Time Frame
- **Default**: today + 7-day outlook
- **If the user specifies a window** ("just today", "this weekend", "next
  Tuesday"): honor that scope. Note if it extends beyond reliable 7-day range.

---

## Step 2: Fetch Data from Multiple Sources

Fetch in parallel where possible — each source serves a different purpose.
For dew point specifically: NWS tabular (Source B) for days 1-2, WeatherBug
hourly (Source C) for days 1-7 — the two overlap early on and together
cover the full default window with no gap.

### Source A: NWS Point Forecast
`https://forecast.weather.gov/MapClick.php?lat={lat}&lon={lon}`

Provides: 7-day text forecast, wind direction/speed, high/low temps,
current conditions from nearest station, hazard alerts.

### Source B: NWS Tabular Hourly Forecast (dew point, days 1-2)
`https://forecast.weather.gov/MapClick.php?lat={lat}&lon={lon}&unit=0&lg=english&FcstType=digital`

Provides: hourly Dewpoint (°F), temp, RH%, wind, sky cover, precip chance —
for a ~48-hour window from NWS's raw forecast grid. This is the **primary
dew point source for today and tomorrow**: it's more precise (hourly, not
just per day/night period), always current (no caching lag), and comes
straight from NWS rather than a repackaged third-party feed. Use this
first for near-term dew point.

### Source C: WeatherBug Hourly Forecast (dew point, days 3-7)
For **Lavallette NJ**: `https://www.weatherbug.com/weather-forecast/hourly/lavallette-nj-08753?day={today|tomorrow|3|4|5|6|7}`
For **Katonah NY**: `https://www.weatherbug.com/weather-forecast/hourly/katonah-ny-10536?day={...}`
For **other locations**: construct as `https://www.weatherbug.com/weather-forecast/hourly/{city}-{state}-{zip}?day={...}`

Provides: dew point per hour, for each of the next 7 days (day tabs:
today, tomorrow, 3, 4, 5, 6, 7). Use this — not the 10-day summary page —
for dew point beyond NWS's 48-hour tabular window. The 10-day *summary*
page (`.../10-day-weather/...`) only surfaces an explicit dew point figure
starting around day 5-6 in its text blurbs, so don't rely on it for dew
point. Don't reach for it at all otherwise — Source A (NWS) already
covers daily high/low/condition/rain chance for the full week, so there's
no remaining reason to fetch the WeatherBug 10-day summary page.

**No coverage gap**: NWS tabular (days 1-2) + WeatherBug hourly (days 1-7,
overlapping and extending past NWS) together give a forecasted dew point
for every day in the default 7-day window. Only fall back to overnight
low + wind direction/air mass reasoning beyond day 7, or if a fetch fails.

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

## Step 3: Build the Forecast Response

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

## Step 4: Response Format

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

---

## Key Reminders

- Dew point source hierarchy: NWS tabular hourly for days 1-2 (primary,
  most precise) → WeatherBug **hourly** page for days 1-7 (use day tabs,
  not the 10-day summary page, which lacks dew point until day 5-6) →
  overnight-low/wind reasoning only beyond day 7 or on a fetch failure
- NWS is also the structure, wind, and pattern source throughout
- NWS AFD explains *why* and *when* the pattern changes — use it for context
- Dew point is ground truth; wind direction is the leading indicator of trend
- Best beach days: post-frontal NW, dew point <63°F, deep blue sky
- Worst beach days: SW wind, dew point 70°F+, overnight low near 75°F+
- The overnight low progression across the week is the clearest signal of
  whether the air mass is drying out or building moisture
