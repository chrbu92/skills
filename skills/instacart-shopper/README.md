# instacart-shopper

A Claude skill that turns a grocery list into a populated Instacart cart, ready for review and checkout. Hands-off after upfront clarification.

## What it does

1. Parses your grocery list (text, markdown, or JSON).
2. Asks any clarifying questions UPFRONT (store, ambiguous items, load-bearing quantities).
3. Opens Instacart in a browser. On the first run, you log in once; subsequent runs reuse the session.
4. Scrapes your "Buy It Again" history to inform substitutions and defaults.
5. Shops fully hands-off — searching, picking matches, adding to cart, substituting when needed.
6. Leaves the browser open on the cart with a clear summary of items added, substitutions made, and items skipped.

## Prerequisites

- **Claude Code** (or any Claude harness that supports the skill format).
- **Playwright MCP server** installed and available. See https://github.com/microsoft/playwright-mcp for installation. The skill checks for it at start and fails fast if missing.
- **An Instacart account** — you'll log in once during the first run.

## Install

### Option A — full plugin
From the parent repo (`cbulloss-skills`):
```
/plugin install <git-url>
```

### Option B — copy this skill alone
```bash
cp -r skills/instacart-shopper/ ~/.claude/skills/instacart-shopper/
# or to a single project:
cp -r skills/instacart-shopper/ <project>/.claude/skills/instacart-shopper/
```

## Usage

Invoke from a Claude Code session:
```
/instacart-shopper
```
Then provide your grocery list (one of the formats below). Or invoke implicitly:
```
Use the instacart-shopper skill with this list:
- 2 lb chicken breast
- milk
- salsa
```

### Input formats

#### Plain text (`.txt` file or inline)

One item per line. Blank lines and `#` comments are ignored. Quantity prefixes parse:
```
2 lb chicken breast
milk
salsa
1 loaf sourdough bread

# this comment is ignored
2 dozen eggs
```

#### Markdown (`.md` file)

Bullet or checklist syntax. Checkbox state (checked or unchecked) is ignored — every line becomes an item:
```markdown
# Grocery list

- [ ] 2 lb chicken breast
- [ ] milk
- [x] salsa
- 1 loaf sourdough bread
```

#### Structured JSON (`.json` file or inline JSON)

Validated against `schemas/grocery-list.schema.json`:
```json
{
  "store": "Wegmans",
  "items": [
    { "name": "chicken breast", "quantity": 2, "unit": "lb" },
    { "name": "milk" },
    { "name": "salsa", "notes": "medium heat" }
  ]
}
```
This format is the natural output of an upstream meal-planning skill. Providing `store` here lets the skill skip the store-selection question.

See [`examples/`](./examples/) for runnable samples of each format.

## What runs interactively vs hands-off

**Interactive (upfront only):**
- Store selection — only if not provided in JSON input.
- Browser login — only on first run, or when the session has expired.
- Clarification batch — only if the list has ambiguous items or load-bearing missing quantities. Up to 4 questions, all in a single `AskUserQuestion` call.

**Hands-off:**
- Everything from "shopping starts" to "browser left open on cart". Substitution decisions are autonomous and reported in the final summary.

If the skill ever prompts mid-shop, that's a bug — file an issue.

## Substitution policy

When the requested item isn't available exactly:
1. **Previously-purchased match.** If you've bought a close equivalent before (per Instacart's "Buy It Again"), use it.
2. **Top Instacart match.** Fall back to the highest-ranked search result that's a reasonable match.
3. **Cheapest viable.** If multiple matches are equivalent, pick cheapest.
4. **Simplify and retry** up to twice (drop adjectives, then keep the head noun).
5. **Skip** if nothing reasonable matches; report in the summary.

## State / data location

The skill writes runtime state to `~/.claude/skills-data/instacart/`:
- `profile/` — persistent Playwright user-data-dir (your login lives here).
- `runs/<timestamp>.json` — optional per-run log of what was added/substituted/skipped.

Nothing is written inside this skill directory. Delete `~/.claude/skills-data/instacart/profile/` to force a fresh login.

## Failure modes

The skill stops with a clear, actionable message in these cases:

- **Captcha / verification challenge** — Solve it in the open browser, then re-run.
- **Session expired mid-run** — Log in again in the open browser, then re-run.
- **Instacart UI change** — A required landmark (search bar, cart, etc.) is missing; the skill reports which one. File an issue.
- **Playwright MCP missing** — Install Playwright MCP and re-run.

The skill does NOT silently retry, auto-solve, or paper over these conditions.

## Limitations

- **Single store per run.** Switching stores mid-list isn't supported.
- **Captcha / 2FA** breaks the hands-off contract — by design, the skill stops and asks you to handle it.
- **Buy-It-Again accuracy** is bounded by what Instacart shows — typically the last 50–100 items per store.
- **Quantity precision** — for items priced by weight (deli meat, produce by lb), Instacart's UI may not allow exact decimal quantities; the skill picks the closest available step.

## Future skills in this family

A planned `amazon-fresh-shopper` skill will follow the same shape (persistent profile, hands-off, single upfront clarification). It will live as a sibling skill, not a fork of this one.
