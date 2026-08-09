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

This skill combines NWS (structure, wind, pattern context) with WeatherBug
(explicit dew point forecasts) to deliver a comfort-first weather interpretation.

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

## Step 2: Fetch Data from Two Sources

Fetch both in parallel — they serve different purposes.

### Source A: NWS Point Forecast
`https://forecast.weather.gov/MapClick.php?lat={lat}&lon={lon}`

Provides: 7-day text forecast, wind direction/speed, high/low temps,
current conditions from nearest station, hazard alerts.

### Source B: WeatherBug 10-Day Forecast
For **Lavallette NJ**: `https://www.weatherbug.com/weather-forecast/10-day-weather/lavallette-nj-08753`
For **Katonah NY**: `https://www.weatherbug.com/weather-forecast/10-day-weather/katonah-ny-10536`
For **other locations**: construct as `https://www.weatherbug.com/weather-forecast/10-day-weather/{city}-{state}-{zip}`

Provides: explicit dew point forecasts per day/night period, humidity %,
wind direction per period. This is the primary dew point source.

### Source C: NWS AFD Discussion (when useful)
Philadelphia/NJ office:
`https://forecast.weather.gov/product.php?site=NWS&issuedby=phi&product=AFD&format=CI&version=1&glossary=1&highlight=off`

NY/Westchester office:
`https://forecast.weather.gov/product.php?site=NWS&issuedby=okx&product=AFD&format=CI&version=1&glossary=1&highlight=off`

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
When WeatherBug dew point isn't available:
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
- Use shorthand freely — Sudeep knows the framework: "post-frontal NW flow",
  "synoptic easterly", "Gulf air mass", "dew point in the 50s"
- Always flag the best day(s) of the week explicitly
- At the beach, always comment on whether wind is the real sea breeze or a
  deceptive synoptic flow

---

## Key Reminders

- WeatherBug is the dew point source; NWS is the structure and pattern source
- NWS AFD explains *why* and *when* the pattern changes — use it for context
- Dew point is ground truth; wind direction is the leading indicator of trend
- Best beach days: post-frontal NW, dew point <63°F, deep blue sky
- Worst beach days: SW wind, dew point 70°F+, overnight low near 75°F+
- The overnight low progression across the week is the clearest signal of
  whether the air mass is drying out or building moisture
