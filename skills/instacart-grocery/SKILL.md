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

Lists are usually copied straight from the Alexa app, and **the export format is not stable** — Alexa/the app changes how it renders a copy-paste from one time to the next. Don't assume a single fixed template; instead recognize the pattern generically each time:

- The item name is the actual grocery item text (e.g. "flour tortillas," "bacon," "Deep cereal").
- Everything else is noise to discard: quantity/checked-off flags (e.g. a line like `Idea	1	0`), who-added-it and when (e.g. "Samantha Added 24 days ago," or just "Added 25 days ago" with no name), and UI action labels ("Show search result," "Edit," "Delete"). None of this affects what gets added.
- **Ignore quantity entirely**, however it's represented — the user adjusts quantities in the cart afterward, so every item goes in as a simple add.
- Attribution (who added an item) is usually irrelevant, but keep it in view for Step 3 — occasionally an item's text only makes sense in light of who added it (see below).

If the list arrives in some other format entirely (plain lines, freeform text, a screenshot), apply the same principle: pull out the real item names and drop everything that's just metadata.

## Step 3: Match each item against the product guide

The guide lives in [references/staples.md](references/staples.md) — read it fresh each time, since it's updated as preferences change.

1. Look for a match — exact or close (shorthand, plural/singular, common nickname) — against a guide entry. Use the guide's specific product name as the search query when found.
2. Respect retailer scope: skip a DeCicco-only entry (e.g. the bakery bread) if Stop & Shop is active, and use the retailer-appropriate variant where the guide lists one per store (e.g. black bean brand).
3. If a term is genuinely ambiguous between two guide entries (e.g. plain "mac and cheese" could mean Stouffer's or Beecher's), ask which one rather than guessing.
4. **If the item text isn't actually a product description** — it names a family member (e.g. "Deep cereal" means a cereal for Deep, not a product called that), or describes an occasion/use rather than a thing to buy (e.g. "bread for grilled cheese" — possibly a different bread than the guide's usual bakery loaf) — don't search it literally. Ask the user what specific product is meant, then treat their answer as a fresh item to resolve (guide match, or browse per below).
5. Anything left with no guide match at all is a genuinely new/one-off item (e.g. shampoo, fig jam) — don't guess a brand or grab whichever result comes back first. Hand these off to Step 4's browse path instead of adding them blind.

## Step 4: Add items to the cart

Two different paths depending on how each item resolved in Step 3:

- **Guide match (including a disambiguated one)** — add directly via `quick_add_search_queries` using the guide's specific product name. No need to show options; the product's already known.
- **No guide match** — use `search_products` (not quick-add) against the active retailer's catalog and show the user a few real options (name, brand/variant, size) pulled from actual results — not invented — so they can pick rather than getting whichever match quick-add would have grabbed silently. Once they pick, add that specific product via the `cart` tool's `item_updates` with its `product_id`.

Don't skip anything on the list, and don't add anything that wasn't on it or separately requested.

## Step 5: Review before checkout

Summarize what was added, noting which items matched the guide (and to what product) versus which were picked from browsed options, so the user can catch a bad match. Use the `cart` tool to pull current contents if a fresh read is useful. Never complete checkout without the user's explicit go-ahead — this skill builds and reviews the cart, it doesn't place the order unattended.

## Updating the guide

When a preferred brand changes, or a new shorthand needs mapping (e.g. "ramen" → Nissin Top Ramen), update [references/staples.md](references/staples.md) directly so future lists resolve correctly, rather than only remembering the correction in conversation.
