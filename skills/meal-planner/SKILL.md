---
name: meal-planner
description: Plans the week's dinners (home-cooked, takeout/delivery, or dinner out) by reading the family Google Calendar for existing MEAL-tagged events and evening availability. Make sure to use this whenever the user asks to plan dinners, plan meals, figure out the week's food, fill in the meal calendar, or mentions "meal planning," "what's for dinner this week," or a specific mix of home-cooked vs. eating-out nights. Balances home-cooked meal complexity and takeout vs. dinner-out based on how busy the week looks, and proposes a plan for review rather than writing to the calendar directly.
---

# Meal Planner

Plans dinners for a Monday-Sunday week by reading the family Google Calendar, working around what's already scheduled, and proposing a balanced mix of dinner types for the remaining nights. Always ends with a proposal for the user to review — never creates or edits calendar events without explicit confirmation.

## Why this exists

A week where everyone's slammed shouldn't get the same dinner plan as a quiet week — simple home-cooked meals and takeout make sense when time is short, while more complex cooking and dinner outings fit better when there's breathing room. This skill reads the week's actual calendar load and lets that shape the mix, rather than applying the same fixed rotation every week.

## The three dinner categories

1. **Home-cooked** — one combined list where each meal carries a complexity/cost point value (1 = quick and cheap, 5 = involved and pricier). Lean on higher-point meals for lighter weeks and lower-point meals for busier weeks, rather than picking from separate "simple" and "complex" buckets.
2. **Takeout/delivery**
3. **Dinner out**

Specific options for each category live in [references/meal-options.md](references/meal-options.md) — a list the user maintains and edits over time. Read it each time; don't assume the options or point values are static from a previous run.

Takeout/delivery and dinner-out options are further split by house — Katonah or Lavallette — since the family splits time between the two. See Step 1 for how to pick which list applies.

## Step 1: Read the week

Use the calendar tools to find the family calendar (search `list_calendars` for the family/shared calendar if it's not obvious which one to use — ask the user if there's any ambiguity) and pull all events for the Monday-Sunday week being planned.

- **All-day events — read dates literally, don't convert timezones**: an all-day event has a `date` field (e.g. `"start":{"date":"2026-07-21"}`), not a `dateTime` field. The API often renders this as `2026-07-21T00:00:00Z`, which looks like an instant but is really just a calendar date — converting it through UTC-to-local math (e.g. to America/New_York) shifts it to 8pm the *previous* day and misattributes the event to the wrong night. Always take the date portion as-is. Also remember the `end` date on an all-day event is exclusive — a one-day event spanning just July 21 will show `start: 2026-07-21` / `end: 2026-07-22`, not `end: 2026-07-21`.
- **Already-planned dinners**: any event titled `MEAL: <description>` — match case-insensitively (your calendar may use `Meal:`) — leave these alone, they're already decided. Note which days they cover.
- **Implicit already-planned dinners**: any event whose title contains one of these words (case-insensitive) also counts as already decided, even without the `MEAL:` tag — skip it automatically, same as a tagged `MEAL:` event: "dinner", "bbq", "barbecue", "cookout", "potluck", "supper" (e.g. "Dinner with the Smiths", "57 Dinners", "Bappa BBQ"). Use judgment on ones that don't clearly fit even if a keyword matches (e.g. a meeting titled "BBQ Planning Committee" isn't itself a meal) — note the call when presenting the plan (Step 6) if it's a close one.
- **Recent history**: also pull `MEAL: ...` events from roughly the past 3-4 weeks. Use these to avoid suggesting a repeat of something recently served, especially within the same category.
- **Location for the week**: default to Lavallette for weeks falling in late June through early September, and Katonah otherwise — these date ranges are approximate, not fixed. State this assumption explicitly when presenting the plan (Step 6) so the user can correct it if that particular week is an exception.
- **Kids' activity events**: any event titled like `<Activity> (Kiran)` or `<Activity> (Mia)` — the parenthetical kid's name is what matters, not the activity name. Note which days have one; these rule out dinner out for that night regardless of what time the activity falls at (see Step 2).
- **Deep In Office**: any event titled `Deep In Office` — Deep is working late at the office that day, so rule out dinner out for that night regardless of timing, same as a kids' activity event (see Step 2).

## Step 2: Find the open nights

For every day in the week without an existing or implicit already-planned dinner (Step 1), check for:

- A conflicting calendar event overlapping 6:00-7:30 PM (confirm this window with the user if they've indicated a different one).
- A kids' activity event or a `Deep In Office` event anywhere that evening (see Step 1), even if it doesn't land inside the 6-7:30 window.

Classify each open night:

- **Dinner-window conflict** → this night can't be home-cooked. Prioritize takeout/delivery over dinner out — it fits around a fixed commitment better than a sit-down reservation does. Only lean dinner out if takeout clearly doesn't make sense for that night.
- **Kids' activity or Deep In Office, no dinner-window conflict** → avoid dinner out (shuttling a kid to/from an activity, or Deep working late, doesn't mix well with a sit-down restaurant), but home-cooked or takeout are both fine.
- **Neither** → open for any category, including home-cooked.

## Step 3: Score the week's busyness

For each open day, compute a simple busyness score: count of non-`MEAL` calendar events that day, plus 2 extra points if that day also has a dinner-window conflict (from Step 2). Average the daily scores across the week to classify the week overall:

- **Light week** (low average score) → skew toward higher-point home-cooked meals and dinner out
- **Moderate week** → balanced mix across home-cooked (spanning point values), takeout/delivery, and dinner out
- **Heavy week** (high average score) → skew toward lower-point home-cooked meals and takeout/delivery

There's no fixed numeric cutoff for light/moderate/heavy — use judgment based on the actual event density you see (e.g. a week that's mostly empty evenings vs. one with something most nights), and explain your reasoning when you present the plan so the user can see why you called it that way. This scoring is a first pass — expect to refine the exact thresholds through use.

Also weigh each open day individually, not just the week average: a single unusually packed day within an otherwise light week can still lean toward the lower-effort categories even if the rest of the week is relaxed.

## Step 4: Choose which nights to plan

By default, always plan exactly **4 of the 7 nights**, not all seven — unless the user has asked for a specific count or specific nights this time. Nights left unplanned are simply left off the plan, for the family to decide spontaneously.

Rank nights by how confident the calendar signal is, and take the top 4:

1. Nights with a dinner-window conflict, a kids' activity event, or a Deep In Office event (Step 2) first — these have a clear, calendar-driven reason for a particular category, so they're worth locking in.
2. If fewer than 4 nights have such a signal, fill the remaining slots from the other open nights (no conflict, no activity) using Step 3's busyness judgment even without a hard signal — always land on 4 rather than presenting fewer. Some variety in which ones you pick is fine — there's no need to always land on the same 4 slots for similar-looking weeks.
3. If more than 4 nights have a conflict or activity signal, use judgment to pick the 4 most worth locking in (or ask the user if it's a close call).
4. **Exception**: if already-planned and implicit-already-planned dinners (Step 1) cover more than 3 days, fewer than 4 days may remain at all — in that case just plan whatever's left; don't invent a signal on a day that's already decided.

**Category coverage**: whichever 4 nights get chosen, the calendar signal alone shouldn't be allowed to crowd out a whole category — across the 4, aim for at least one home-cooked, one takeout/delivery, and one dinner-out night, even when conflicts/activities don't naturally produce that mix. Prefer **swapping which nights are in the 4** over overriding a kept night's natural category — it keeps each night's category tied to what the calendar actually says, rather than fighting it:

1. Home-cooked can only happen on an open night (Step 2's hard rule) — make sure at least one of the 4 is open, swapping in an open night for the weakest-signal conflict/activity night if all 4 would otherwise be conflicts.
2. If dinner out is still missing, look for a **second** open night to swap in next (dropping another conflict/activity night, weakest-signal first) rather than reassigning a kept conflict night away from takeout. Only override a kept night's default category — e.g. push a conflict night to dinner out instead of takeout — if there's no open night left available to swap in for it, and only on a night that doesn't also carry a kids'-activity or Deep In Office exclusion (those still can't be dinner out).
3. If it's genuinely impossible to fit all three — e.g. every open day has a kids' activity ruling out dinner out, or already-planned dinners (Step 1) leave too few days to work with — say so explicitly in Step 6 rather than forcing a bad fit.

## Step 5: Propose specific meals for each chosen night

For each of the nights chosen in Step 4, pick a category consistent with Step 2's classification and Step 3's skew, subject to the category-coverage requirement from Step 4 (a night reassigned there to guarantee coverage follows its reassigned category, not Step 2's default):

- **Home-cooked**: pick a meal whose point value matches the skew for that day/week (higher points on lighter days, lower points on busier days, either on moderate days) — avoiding anything served recently per Step 1's history check.
- **Takeout/delivery or dinner out**: pick from the list for the location determined in Step 1 (Katonah or Lavallette) — avoiding recent repeats the same way, and respecting Step 2's dinner-out exclusion on kids'-activity and Deep In Office nights.

Vary choices across the week rather than repeating the same dish or restaurant twice even if it's not in the recent-history window.

## Step 6: Present the plan for review

Show a day-by-day table for the chosen nights only: day, category, specific meal (or "already planned: <description>" for existing events), and a one-line note on why that category was chosen (dinner-window conflict, kids' activity, Deep In Office, busier/lighter day, etc). List the nights left unplanned separately and briefly summarize the week's overall busyness call from Step 3.

**Do not create, edit, or delete any calendar events at this step.** Ask the user if the plan looks right, and only create `Meal: <description>` events (title case, matching the existing calendar convention) for the nights they approve, using their explicit go-ahead for each change (or a clear "yes, add all of these" for the whole batch).
