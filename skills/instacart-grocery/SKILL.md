---
name: instacart-grocery
description: Places the family's recurring Instacart grocery order, auto-detecting whether to shop DeCicco & Sons (Katonah, early September through late June) or Stop & Shop (Lavallette, late June through early September) based on the date. Use this whenever the user asks to order groceries, place the Instacart order, do the weekly/grocery shop, restock the house, or mentions "grocery order," "Instacart," or names either store directly. Builds the cart from a maintained staples list instead of starting from scratch each time — auto-adds well-established staples, confirms occasional/seasonal items, and asks what's different this week before anything goes to checkout.
---

# Instacart Grocery

Builds the family's Instacart cart from a maintained staples list, split by which house/retailer is active, rather than re-deciding what to order from nothing each time. Always ends with a review step — never checks out without the user's explicit go-ahead.

## Why this exists

The family's Instacart orders repeat the same ~20 items week after week, with a smaller set of occasional or seasonal add-ons on top. Retyping the whole list every time is wasted effort, but silently auto-ordering everything risks a cart full of stale defaults the user didn't actually want that week. This skill splits the difference: the well-established staples go in automatically, the occasional/seasonal tier gets a quick confirm, and there's always a slot for whatever's different this time.

## Step 1: Determine retailer and season

Use the same seasonal split as the meal-planner skill: default to **Stop & Shop (Lavallette)** for late June through early September, and **DeCicco & Sons (Katonah)** otherwise. These date ranges are approximate, not fixed — state the assumption explicitly so the user can correct it if this particular week is an exception (e.g. an off-season trip to the beach house).

## Step 2: Select the retailer

Call the `cart` tool with `retailer_name` set to the retailer from Step 1. If a different retailer is already active in the session, confirm with the user before switching.

## Step 3: Build the cart from staples

The staples list lives in [references/staples.md](references/staples.md) and is maintained by the user — read it fresh each time rather than assuming it hasn't changed since last use.

1. **Core tier** — items ordered constantly regardless of which store is active. Add these automatically via `quick_add_search_queries`, no confirmation needed. A couple of core items rotate brand/variety order to order (e.g. cereal) rather than repeating one fixed SKU — for those, ask a quick one-line "which one this week?" using the options listed in staples.md instead of guessing, then add the chosen one.
2. **Regular-but-infrequent tier** — items that are clearly habits, not one-offs, but don't appear in every order and aren't tied to a specific store. These apply regardless of which house is active. Read them out as a quick checklist ("want any of the usual extras — Beecher's, chicken breast, ...?") and add only what the user confirms — don't add the whole tier by default.
3. **Retailer-specific tier** — currently just one item: the bakery bread (Frank Zurro's/Terranova Pane Di Casa) that's only good at DeCicco & Sons. Add it automatically when DeCicco is the active retailer; skip it entirely at Stop & Shop rather than substituting something else, since there isn't a comparable bread there.
4. **Seasonal/occasional tier** — items bought periodically based on time of year or occasion, not tied to either store (e.g. winter ice-melter, taco-night kits, a specific protein). Read these out as a short list and ask which, if any, to include this order — don't add without confirmation.

Most of the regular-but-infrequent tier is presumed to apply at Stop & Shop too, even where the small 2-order sample hasn't confirmed it directly yet.

## Step 4: Ask what's different this week

After the staples pass, ask one open question — something like "anything else for this order?" — for whatever doesn't repeat cleanly: a specific protein, ingredients for a planned meal (this pairs naturally with the meal-planner skill's output), produce that varies by season, or a one-off item. Add whatever the user names via `quick_add_search_queries`, using `search_products` first if they want to browse/compare options rather than grab the first match.

## Step 5: Review before checkout

Summarize what went into the cart — grouped by core / retailer-specific / seasonal / this-week's-extras — so the user can catch anything wrong before checkout. Use the `cart` tool to pull current contents if a fresh read is needed. Never complete checkout without the user's explicit go-ahead; this skill builds and reviews the cart, it doesn't place the order unattended.

## Updating the staples list

Staples drift over time — a new item becomes routine, an old one falls out of rotation. When the user mentions a changed habit, or after reviewing a fresh batch of order history, update [references/staples.md](references/staples.md) directly so the change sticks for future orders, rather than only tracking it in conversation.
