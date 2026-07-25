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

## Step 5: Pick a recommendation for each chosen night

For each of the nights chosen in Step 4, settle on **one recommended option** consistent with Step 2's classification and Step 3's skew, subject to the category-coverage requirement from Step 4 (a night reassigned there to guarantee coverage follows its reassigned category, not Step 2's default):

- **Home-cooked**: a meal whose point value matches the skew for that day/week (higher points on lighter days, lower points on busier days) — avoiding anything served recently per Step 1's history check.
- **Takeout/delivery or dinner out**: an option from the list for the location determined in Step 1 (Katonah or Lavallette) — avoiding recent repeats the same way, and respecting Step 2's dinner-out exclusion on kids'-activity and Deep In Office nights.

Vary recommendations across the week rather than recommending the same dish or restaurant on multiple nights, even if it's not in the recent-history window.

The recommendation is what leads each night's list in Step 6 — it isn't a locked-in choice. Also note that night's full set of same-category options from [references/meal-options.md](references/meal-options.md), since the widget presents all of them.

Give each option a food/cuisine/restaurant-flavored emoji picked on the spot — there's no fixed mapping to look up. Mix it up: the obvious, recognizable pick is often the right call (🍔 for burgers, 🍕 for pizza, a national flag like 🇮🇹 for an Italian spot), but don't feel locked into it every time — an occasional more playful or unexpected choice (🐟 or 🥢 for a sushi night, say) is welcome too. Just keep whatever you pick clearly connected to the actual dish or place.

## Step 6: Let the user choose

Offer the options per night and let the user pick — including room to type something not on any list at all.

**Preferred: an interactive widget**, when a tool for rendering interactive HTML is available. The user strongly prefers this over typing out picks by hand. Render a form with one card per chosen night (day, category, and the one-line reason), each holding a dropdown plus a single submit button that compiles every night's pick into one message. Keep the broader explanation in your normal response text, not inside the widget itself.

Structure each dropdown in three `<optgroup>`s, listing **every** option in that night's category — a collapsed dropdown costs no space, so there's no reason to truncate:

1. **"Recommended"** — the single Step 5 pick, and the dropdown's pre-selected default, so agreeing means just hitting confirm.
2. **"All &lt;location&gt; &lt;category&gt;"** (e.g. "All Lavallette takeout", "All home-cooked") — every remaining option in that category, minus the recommendation itself.
3. **"Other"** — "Something else..." (reveals a free-text input) and "Skip this night".

Keep option labels clean: emoji plus the name, nothing else. Don't append point values, "had last week" tags, or other annotations — that curation is what the recommendation is for.

Requirements learned from real use — follow these or the form silently breaks:

- **Scope every element lookup to a uniquely-named root** (e.g. `<div id="mealplan-<week>">`) and find children via `data-` attributes, not shared `id`s. Earlier widgets from the same conversation stay in the DOM; reusing ids like `submit-btn` makes `document.getElementById` return a stale element from a previous widget.
- **Wrap the script in an IIFE.** Top-level `const`/`let` at global scope collide with identically-named declarations from earlier widgets and throw a SyntaxError that kills the whole script before any listener attaches.
- **Include a small status line** under the button that updates on click ("Sending…" → "Sent."). Without it there's no way to tell a dead handler from a host-side drop.
- **How submission actually works**: the send function injects the composed text into the user's chat composer, which they then send themselves — it does not post a message on its own. The injection only lands when the composer is clean. If the user has typed in or deleted text from the composer, it's "dirty" and further injections are silently dropped even though the call reports success. Sending any message resets it to clean; deleting the text does not.
- **Always reveal the composed text on submit, in a readonly input with a copy button** — never rely on the injection alone. Because a refused injection looks identical to a successful one from inside the widget, this is the only thing that keeps the user from being stuck with a dead button and no way to proceed. Word the status line to cover both cases ("Sent to composer — press enter. If nothing appeared there, copy the text below instead.").
- To revise after submitting, render a *fresh* widget rather than asking the user to re-click the old one.

**Fallback — use whenever no widget tool is available**: plain text. Here the full-list approach doesn't work — nine options a night is unreadable — so show the Step 5 recommendation plus **2-3 alternates** per night instead. Show a day-by-day table: day, category, numbered options with emoji (or "already planned: <description>" for existing events), and a one-line note on why that category was chosen (dinner-window conflict, kids' activity, Deep In Office, busier/lighter day, etc). List nights left unplanned separately, and summarize the week's overall busyness call from Step 3. Ask the user to reply with a pick per night — by number, by naming something else, or by saying to skip a night.

Either way, still state the reasoning, the unplanned nights, and the week's busyness call in your response text.

**Do not create, edit, or delete any calendar events at this step** — this step only presents choices.

## Step 7: Create the events

Once the user responds — via the widget submission or a typed reply — treat that as the approval. For each night with a real pick, create a `Meal: <emoji> <description>` event (title case, matching the existing calendar convention, emoji right after the `Meal:` prefix; skip the emoji only if nothing reasonable fits). Default to a 6:00-7:00 PM slot unless the user says otherwise — it matches the existing events' timing and sits in the dinner window.

Create nothing for any night marked "skip", left blank, or not addressed.

If anything is ambiguous, confirm that one night rather than guessing — and note that a custom free-text entry that doesn't read like a meal (e.g. an obvious placeholder or test string) is worth checking before writing it to a shared family calendar.
