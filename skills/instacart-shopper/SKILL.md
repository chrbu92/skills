---
name: instacart-shopper
description: Build an Instacart cart from a grocery list. Use when the user provides a list of grocery items (text, markdown, or JSON) and wants a cart populated and ready to check out. Asks all clarifying questions upfront, then runs hands-off through shopping.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(mkdir *)
  - Bash(ls *)
  - AskUserQuestion
  - mcp__playwright__*
---

# instacart-shopper

Turn a grocery list into a populated Instacart cart, ready for the user to review and check out. Designed to be **hands-off**: the user provides input, answers clarifying questions upfront, completes login once, then walks away.

## The hands-off contract

This is the skill's most important property. Read carefully:

1. **All clarification happens BEFORE shopping starts.** Use `AskUserQuestion` only during the parse + setup phases (Phases 1–4 below).
2. **No `AskUserQuestion` calls during shopping (Phase 5).** Substitution decisions are made autonomously and reported in the summary.
3. **If something blocks the skill mid-run** (captcha, expired session, unexpected UI), STOP, surface a clear actionable message, and exit. Do not silently wait, retry forever, or paper over.

If a future change to this skill would add a mid-shop user prompt, that change is **wrong** and must be redesigned.

## Prerequisites

- **Playwright MCP server** must be installed and available as `mcp__playwright__*`. If it is not, fail at the top with: "Playwright MCP server is required. Install it (see https://github.com/microsoft/playwright-mcp) and re-run."
- **An Instacart account** — the user will log in once during the first run; the session persists.

## State / data layout

| Path | Purpose |
|---|---|
| `~/.claude/skills-data/instacart/profile/` | Persistent Playwright user-data-dir. Login persists here across runs. |
| `~/.claude/skills-data/instacart/runs/<timestamp>.json` | Optional run log: items requested, added, substituted, skipped. Useful for debugging. |

Create these directories with `mkdir -p` at the start of the run if they don't exist. **Never** write state inside the skill directory.

## Workflow

### Phase 1 — Parse input

The user provides the grocery list in one of three ways:

**Inline text** — items separated by newlines or commas in the user's message.

**File path** — a `.txt`, `.md`, or `.json` file. Detect by extension.
- `.txt` → plain-text list (one item per line, blank lines and `#` comments ignored).
- `.md` → markdown checklist or bullet list. Strip `- [ ]`, `- [x]`, `- `, `* `, `+ ` prefixes. Ignore checkbox state.
- `.json` → validate against `schemas/grocery-list.schema.json` (sibling to this file).

**Inline JSON** — if the message contains a JSON object matching the schema, treat it as structured input.

Normalize every entry into:
```ts
{ name: string, quantity?: number, unit?: string, notes?: string, raw: string }
```
- Parse leading quantities (`"2 lb chicken"` → `{name: "chicken", quantity: 2, unit: "lb"}`).
- Parse trailing parentheticals as notes (`"salsa (medium heat)"` → `{name: "salsa", notes: "medium heat"}`).
- Dedup case-insensitively on `name + unit`. Sum quantities on dedup if numeric.
- Drop empty lines and lines starting with `#`.

If the input is empty after parsing, fail: "No items found in the grocery list."

### Phase 2 — Pre-browser clarification (store)

If the input doesn't specify a store, ask:
- Single `AskUserQuestion`: "Which Instacart store should I shop at?"
- Provide 2–4 sensible options if you can infer them (common chains: Wegmans, Food Lion, Harris Teeter, Costco). Otherwise let the user type via "Other".

If a store IS specified in the input, skip this phase entirely.

This is the only question that fires before the browser opens — it's needed to navigate to the right store page during setup.

### Phase 3 — Browser setup and login (interactive only on first run / expired session)

1. `mkdir -p ~/.claude/skills-data/instacart/profile/`.
2. Launch browser via `mcp__playwright__browser_navigate` to `https://www.instacart.com/`. The Playwright MCP should be configured to use the persistent profile dir above. (If your Playwright MCP doesn't expose user-data-dir control, document the limitation in the run summary — login will not persist.)
3. Take a `browser_snapshot` to inspect login state. Detect "logged in" by presence of the account menu / cart count, or "logged out" by presence of "Log in" / "Sign up" buttons.
4. **If logged out**, message the user clearly:
   > "I've opened Instacart in a browser. Please log in and select store '<store>'. Reply 'ready' (or 'done') when you're on the store's main shopping page."
   Then wait for the user's response. Do NOT auto-enter credentials. Do NOT poll silently — wait for the explicit text confirmation.
5. **If logged in**, navigate to the chosen store's shopping page. Common URL pattern: `https://www.instacart.com/store/<store-slug>/storefront`. If you can't infer the slug, search Instacart's storefront list using `browser_type` in the search/store-finder.

### Phase 4 — Scrape "Buy it again" + post-browser clarification

1. Navigate to the user's "Buy it again" / "Your items" / order history view for the selected store. The path varies; common patterns:
   - `/store/<store>/your-items`
   - `/store/<store>/buy-it-again`
   - The "Your items" tab on the store page.
2. Use `browser_snapshot` (NOT a screenshot) to extract the previously-purchased list as structured data. Parse into:
   ```ts
   PrevItem = { name, brand?, size?, lastQty?, productUrl? }
   ```
   Capture as many entries as fit comfortably in context (target 50–100). If pagination is needed, scroll once or twice.
3. **Build the post-browser clarification batch.** Identify items in the parsed list that need clarification:
   - **Ambiguous items** — bare names like "snacks", "cheese", "pasta sauce", "yogurt", "bread" with no specifier. For each, suggest 2–3 options sourced from `PrevItem` matches in that category, plus an "Other" fallback.
   - **Load-bearing missing quantities** — items where quantity changes the order materially: proteins (chicken, beef, fish, etc.), produce by weight (apples, bananas, potatoes), bulk items. For each, suggest 2–3 quantity options biased to the user's typical previous purchase (`lastQty`).
4. Issue a single `AskUserQuestion` call with up to 4 questions, prioritized by impact (proteins/produce quantities first, then most-ambiguous categories). If more than 4 clarifications would be needed, batch the top 4 and let the rest default — note the deferred items in the final summary.
5. After the user answers, you have a **finalized normalized list**. Print a one-line confirmation: "Got it. Shopping for N items at <store>. I'll be done in a few minutes." Then start Phase 5 immediately. **No further questions.**

### Phase 5 — Shopping (HANDS-OFF)

For each item in the finalized list, in order:

1. **Try Buy-It-Again match first.**
   - Fuzzy-match `item.name` against the in-memory `PrevItem` list on `name` (lowercased, stripped of articles), then check `unit` and `size` for compatibility.
   - If a confident match exists: navigate to that product (`productUrl`), click "Add" / set quantity to `item.quantity || 1`, and verify cart count incremented via snapshot.
   - Record as `added` (not substituted), continue to next item.

2. **Otherwise, search.**
   - Click the search bar (`browser_click` on the search input by accessible name).
   - `browser_type` the item name. Press Enter.
   - `browser_snapshot` the results region.
   - Pick the best match by these criteria, in order:
     1. Top-ranked / popularity (Instacart ranks results by relevance; first 1–3 results are usually correct).
     2. Cheapest as a final tiebreaker if the top results are clearly equivalent.
   - Confirm the result is reasonably close to the request (name overlap). If yes: add to cart.

3. **If no usable result, simplify and retry** (max 2 simplification rounds):
   - Round 1: drop adjectives (e.g., "organic baby spinach" → "baby spinach").
   - Round 2: keep only the head noun (e.g., "baby spinach" → "spinach").
   - If a simplified search succeeds, the item is **substituted** — record `original_request` and `actual_item`.

4. **If still no match**, mark `skipped` with reason `"no match found"` and continue.

5. **Track every item** in this shape:
   ```ts
   {
     requested: { name, quantity, unit, notes },
     status: 'added' | 'substituted' | 'skipped',
     actual: { name, brand, size, qty, price } | null,
     reason?: string,
     fromBuyItAgain: boolean
   }
   ```

**Hard rules during this phase:**
- Do NOT call `AskUserQuestion`.
- Do NOT prompt the user via printed text and wait for input.
- If you hit a captcha or login redirect, **stop** and surface the failure (see "Failure modes" below).
- If a single item takes more than ~3 attempts without progress, mark it `skipped` with reason `"could not add — see logs"` and move on. Don't get stuck.

### Phase 6 — Report

1. Navigate the browser to the cart page so the user can review immediately. Common URL: `https://www.instacart.com/store/<store>/cart`. If unsure, click the cart icon in the snapshot.
2. **Leave the browser open.** Do NOT call `browser_close`.
3. Print a structured text summary to the user:

   ```
   ## Instacart shopping complete — <store>

   Browser is open on your cart for review.

   ### Added (<N>)
   - 2 lb chicken breast — Tyson, Boneless Skinless ($X.XX)
   - 1 carton milk — Horizon Organic 2%, 1 gal ($X.XX)
   - ...

   ### Substituted (<M>)
   - "organic baby spinach" → Earthbound Farm Baby Spinach, 5oz — reason: no organic option in stock
   - ...

   ### Skipped (<K>)
   - "kombucha (lavender flavor)" — reason: no match found
   - ...

   ### Defaulted without asking
   (Items where quantity or specifics were inferred — adjust in the cart if needed.)
   - "yogurt" → defaulted to 1× Chobani Greek Whole Milk Plain (your last purchase)
   ```

4. Optionally write the run log to `~/.claude/skills-data/instacart/runs/<ISO-timestamp>.json` for debugging.

## Failure modes (and what to do)

| Condition | Action |
|---|---|
| Playwright MCP not available | Fail at top with installation instructions. Do not start the run. |
| Empty grocery list after parsing | Fail with message: "No items found in the grocery list." |
| User aborts during login wait | Treat as cancellation; do not run shopping. |
| Captcha / verification challenge mid-run | STOP. Surface: "Instacart is showing a verification challenge. Solve it in the open browser and re-run the skill." Do NOT retry. |
| Session expires mid-shop (redirect to login) | STOP. Surface: "Session expired mid-shop. Log in again and re-run." Do NOT attempt to log back in autonomously. |
| Expected page landmark missing (search bar, cart icon, results region) | STOP. Surface the exact missing landmark and the URL. Likely an Instacart UI change — don't paper over. |
| A single item gets stuck (>3 add attempts) | Mark `skipped` with reason and continue. Don't block the run. |
| Browser crash | STOP. Surface the error. Do not auto-restart. |

## Known interruption points

This skill IS hands-off after Phase 4 ends. The only required user interactions happen UPFRONT:
- Phase 2 store question (skipped if store provided in input).
- Phase 3 login + store selection (skipped if profile already authenticated).
- Phase 4 clarification batch (skipped if list was already unambiguous and quantities were specified).

Once Phase 5 starts, no user input is requested or required. If the skill ever asks the user a question between "shopping started" and "summary printed", that's a regression — file a bug.

## Implementation notes for Claude

- **Prefer `browser_snapshot` over screenshots** for parsing. Screenshots are useful for verification and final confirmation; the accessibility tree is more reliable for extracting structured data like prices, product names, and button states.
- **Use `browser_evaluate` sparingly** — only when an element is hard to address via accessible role/name. Prefer `browser_click` with semantic selectors.
- **Watch the cart count** in the page header before/after each "Add" — if it doesn't increment, the click missed; retry once.
- **Don't burn context on screenshots** during the shopping loop. Snapshot the relevant search-results region only, not the whole page.
- **Be careful with sized vs. unsized items.** Some items require selecting a size (gallon vs. half-gallon milk). Default to the most-purchased size if seen in `PrevItem`; otherwise, the cheapest in-stock size.
- **Quantity field** — Instacart usually exposes a "+/-" stepper after Add. Click "Add" first (which sets qty=1), then click "+" `quantity - 1` times. Alternative: type into the quantity input if available.
