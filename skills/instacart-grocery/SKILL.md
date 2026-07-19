---
name: instacart-grocery
description: Builds the family's Instacart cart from a pasted grocery list (e.g. copied from the shared Alexa shopping list) by matching each item against a maintained preferred-products guide, instead of typing out full product names and brands every time. Use this whenever the user pastes or types a shopping list and wants it turned into an Instacart order, asks to place the grocery/Instacart order, mentions the Alexa list, "Beach House" or the Katonah house list, or names either store (DeCicco & Sons / Stop & Shop) directly. Adds only what's on the pasted list or explicitly requested — never adds items unprompted just because they're usual.
---

# Instacart Grocery

Turns a pasted shopping list into an Instacart cart, using a maintained product guide to resolve shorthand ("bacon," "ramen," "chips") into the family's actual preferred brand/product, instead of guessing cold or asking about every single item. Adds only what's on the list (or what the user explicitly asks for) — nothing goes in automatically just because it's a "usual" item.

## Why this exists

The family manages groceries with a shared Alexa list — easy to add to by voice, but it doesn't translate cleanly into specific Instacart products ("bacon" could mean several things). This skill bridges that gap: paste the list, and each item gets matched against a maintained guide of preferred brands built from past orders and corrections, then added to the cart for review.

## Step 1: Figure out the retailer

- If the pasted list carries a household name (e.g. "Beach House"), use that directly — Beach House → Stop & Shop (Lavallette); the Katonah-side list → DeCicco & Sons.
- If there's no name on the list to go by, fall back to the seasonal default used elsewhere in this repo: Stop & Shop (Lavallette) for late June through early September, DeCicco & Sons (Katonah) otherwise.
- State whichever assumption is being made and let the user correct it.

Call the `cart` tool with `retailer_name` set accordingly. If a different retailer is already active in the session, confirm before switching.

## Step 2: Parse the pasted list

Lists are often copied straight from the Alexa app, which exports in a two-line-per-item block: the item name (once plain, once quoted), followed by a metadata line like `Idea	1	0` (entry type, quantity, checked-off flag). Extract just the item names. **Ignore quantity entirely** — the user adjusts quantities in the cart afterward, so every item goes in as a simple add regardless of what the metadata line shows.

If the list arrives in some other format (plain lines, freeform text), parse it the same way: one grocery item per entry, ignoring any quantity mentioned.

## Step 3: Match each item against the product guide

The guide lives in [references/staples.md](references/staples.md) — read it fresh each time, since it's updated as preferences change.

1. Look for a match — exact or close (shorthand, plural/singular, common nickname) — against a guide entry. Use the guide's specific product name as the search query when found.
2. Respect retailer scope: skip a DeCicco-only entry (e.g. the bakery bread) if Stop & Shop is active, and use the retailer-appropriate variant where the guide lists one per store (e.g. black bean brand).
3. If a term is genuinely ambiguous between two guide entries (e.g. plain "mac and cheese" could mean Stouffer's or Beecher's), ask which one rather than guessing.
4. If there's no guide match at all, use the user's own wording as the search query directly — don't invent a brand.

## Step 4: Add everything to the cart

Add every item from the pasted list via `quick_add_search_queries` — one entry per item, using the resolved product name from Step 3 (or the original wording if unmatched). Don't skip anything on the list, and don't add anything that wasn't on it or separately requested.

## Step 5: Review before checkout

Summarize what was added, noting which items matched the guide (and to what product) versus which were searched as-typed, so the user can catch a bad match. Use the `cart` tool to pull current contents if a fresh read is useful. Never complete checkout without the user's explicit go-ahead — this skill builds and reviews the cart, it doesn't place the order unattended.

## Updating the guide

When a preferred brand changes, or a new shorthand needs mapping (e.g. "ramen" → Nissin Top Ramen), update [references/staples.md](references/staples.md) directly so future lists resolve correctly, rather than only remembering the correction in conversation.
