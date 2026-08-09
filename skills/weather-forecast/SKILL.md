---
name: weather-forecast
description: >
  Pull a live, personalized weather forecast with special emphasis on perceived
  comfort and humidity. Use this skill whenever the user asks about the weather,
  forecast, how humid it is, whether it's a good beach day, or any question
  about current or upcoming conditions — at their current location OR any
  location they mention. Always fetch live NWS data; never answer from memory.
  Trigger on casual phrasing: "how's the weather", "nice out?", "beach weather
  this week?", "is it going to be sticky?", "good day to be outside?",
  "what's it like in [place]?", "will it be humid this weekend?".
---

# Weather Forecast Skill

## Overview

This skill fetches live NWS data and interprets it through a comfort-first
lens — with special attention to humidity perception, dew point, and
(when at the coast) wind direction and air mass origin.

Sudeep's known locations for context:
- **Lavallette NJ** (beach house) — barrier island, Jersey Shore
- **Katonah NY** (home) — inland Westchester

---

## Step 1: Determine Location and Time Frame

### Location
- **If the user names a specific location** ("what's the weather in Boston?",
  "how's it looking in Katonah?"): use that location. Note it explicitly in
  your response ("Here's the forecast for Boston...").
- **Otherwise**: call `user_location_v0` with `accuracy: precise` to get
  current location, and use that silently without narrating the lookup.

### Time Frame
- **Default**: today + 7-day outlook
- **If the user specifies a different window** ("just today", "this weekend",
  "next Tuesday", "the next two weeks"): honor that scope. Note when the
  requested window extends beyond NWS's reliable 7-day range.
- Flag clearly if you're showing a non-default time frame so the user knows
  what they're getting.

---

## Step 2: Fetch NWS Data

Construct the NWS point forecast URL from coordinates:
`https://forecast.weather.gov/MapClick.php?lat={lat}&lon={lon}`

For the Philadelphia/NJ region (Lavallette), also fetch the AFD discussion:
`https://forecast.weather.gov/product.php?site=NWS&issuedby=phi&product=AFD&format=CI&version=1&glossary=1&highlight=off`

For other regions, the NWS office varies — use the point forecast page which
links to the correct local office's discussion. Fetch the AFD when pattern
context is useful (fronts, air mass changes, multi-day outlook).

For dew point data, also fetch the tabular forecast when available:
`https://forecast.weather.gov/MapClick.php?lat={lat}&lon={lon}&unit=0&lg=english&FcstType=digital`

---

## Step 3: Build the Forecast Response

### Always cover (adjusted to requested time frame):
1. **Today** — conditions, high, wind direction/speed
2. **Tonight** — overnight low (key humidity signal)
3. **7-day outlook** — flag best and worst days for comfort

### Always emphasize comfort/humidity using this framework:

#### Dew Point is the Ground Truth
| Dew Point | Comfort Level |
|---|---|
| Below 55°F | Crisp, refreshing |
| 55–60°F | Comfortable |
| 60–65°F | Acceptable, slightly humid |
| 65–68°F | Middle ground — other factors decide |
| 68–70°F | Muggy, uncomfortable |
| 70°F+ | Oppressive — no real relief regardless of wind |

#### Overnight Low as Humidity Proxy
When dew point isn't explicitly available:
- Low drops to low-60s or below → dry air mass, comfortable
- Low stays at 70°F+ → saturated air mass, no overnight relief

#### Wind Direction at the Coast (apply when near ocean/beach)
| Wind | Comfort Effect | Why |
|---|---|---|
| NW, N | Very good | Post-frontal Canadian air, dry |
| NE (post-frontal) | Good | Dry air wrapping around high pressure |
| E/SE (thermal, afternoon) | Good | Local land-driven breeze, enhances evaporation |
| E/SE (synoptic, summer) | Deceptive — often humid | Warm Atlantic fetch, moisture-loaded |
| S, SW | Bad | Gulf moisture or humid continental air |
| W (post-frontal) | Neutral to good | Drier than SW |

**Key distinction for summer**: A large-scale easterly in July–August is NOT
the refreshing sea breeze — it's been sitting over warm Atlantic water (low-to-
mid 70s°F) and picks up moisture the whole way. The genuinely refreshing
easterly is a local, shallow, afternoon thermal effect driven by land heating,
or a post-frontal pattern.

#### Air Mass Origin (most important factor at the beach)
- **Post-frontal NW/N flow** = dry Canadian air = best days
- **SW flow** = Gulf moisture = worst days
- **Synoptic E/SE in summer** = warm Atlantic moisture = deceptively humid
- **Stagnant/light winds** = moisture builds locally

#### Inland vs. Coast
- **Inland** (Katonah, etc.): dew point alone is the primary comfort
  predictor. Wind direction matters less. Straightforward.
- **At the coast** (Lavallette, etc.): dew point + wind direction + air mass
  origin together determine comfort. Same dew point can feel very different
  depending on wind.

#### Front Passage Signals
Read these from the AFD discussion when available:
- "Post-frontal" = good news coming
- "Gulf moisture surging" / "warm front lifting" = bad news
- "Drier airmass by [day]" = flag as relief day
- "Dewpoints mixing out" = afternoon improvement expected
- "Milky/hazy sky" = humid air mass in place regardless of wind

---

## Step 4: Response Format

Lead with a **one-line comfort verdict** for today (or the requested period),
then expand.

```
**[Location] — [Date/Period]**
Today: [comfort verdict + key conditions]
Tonight: [overnight low + what it signals]
This week: [day-by-day, highlight best/worst days]
What to watch: [fronts, pattern shifts, relief days]
```

If the user asked about a **non-default location**, open with:
"Here's the forecast for [place]..." so it's clear you're not pulling
their current location.

If the user asked about a **non-default time frame**, note it:
"Here's just the weekend..." or "Looking at Tuesday specifically..."

Keep it conversational. Sudeep is fluent in this framework — use the
shorthand freely: "post-frontal NW flow", "synoptic easterly", "Gulf
air mass", "dew point in the 50s", "overnight low tells the story".

---

## Key Reminders

- NWS is always the data source — never answer weather from training memory
- Dew point is the ground truth for comfort; wind direction is the leading
  indicator of where dew point is heading
- The NWS AFD (meteorologist discussion) is the best source for understanding
  *why* the pattern is what it is and *when* it will change
- Best beach days: post-frontal NW flow, dew point below 63°F, deep blue sky
- Worst beach days: SW wind, dew point 70°F+, overnight low near 75°F+
