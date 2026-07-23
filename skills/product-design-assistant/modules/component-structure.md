# Component Structure Contract

Structural definition of every component the Starter Kit must produce. This file is
**preset-agnostic and platform-agnostic** — it defines *what exists* (sizes, variants,
states, props), never *how it looks* (that comes from the active preset in
`presets/*.yaml`) or *where things sit* (that comes from `platform-conventions/*.md`).

Keeping this separate from presets means adding a 4th preset later never requires
re-deciding "does Button have a destructive variant" — that question is answered once,
here.

When generating a component, resolve it as:

```
final_spec = component_structure[name]
           + preset.components[name]        (radius/elevation/density values)
           + platform_convention[name]       (placement/native-pattern overrides, if applicable)
```

## Button

- **Sizes**: `sm`, `md`, `lg`
- **Variants**: `primary` (solid), `secondary` (outline or tonal — preset decides), `tertiary` (text-only), `destructive`
- **States**: `default`, `hover` (web only), `pressed`, `disabled`, `loading`
- **Props**: label, optional leading/trailing icon, full-width flag
- **Padding**: horizontal ≥ 1.5× vertical padding (from spacing scale)
- **Radius token**: `radius.button`
- **Usage rule**: exactly one `primary` button visible per screen/section — never two competing primaries

## Input

- **Sizes**: `md`, `lg` (no `sm` — inputs need minimum tap/click target)
- **Variants**: `text`, `number` (uses numeric typography if preset defines it), `search` (see Search), `textarea`
- **States**: `default`, `focused`, `filled`, `error`, `disabled`
- **Props**: label (always visible, never placeholder-only), helper text, error text, leading/trailing icon
- **Radius token**: `radius.input`

## Card

- **Variants**: `default` (surface + border/shadow per preset), `interactive` (adds pressed state, used when card is tappable), `outlined` (no elevation, border only)
- **States**: `default`, `pressed` (interactive only)
- **Padding**: internal padding = `spacing.card` (from preset spacing scale)
- **Radius token**: `radius.card`

## Bottom Navigation (mobile only)

- **Item count**: 3–5 items (never fewer than 3, never more than 5 — overflow goes in a "More" tab)
- **States per item**: `active`, `inactive`
- **Props**: icon (required), label (required — icon-only bottom nav is not permitted, PMs' users need labels)
- **Platform note**: placement/native behavior resolved by `platform-conventions/{ios,android}.md`

## Navigation Bar / App Bar

- **Variants**: `root` (title + optional trailing action, no back button), `detail` (back button + title + optional trailing action)
- **Props**: title, leading action, trailing action(s) — max 2 trailing actions
- **Platform note**: back-button icon/position resolved by platform convention

## Search

- **Variants**: `inline` (embedded in a page), `modal` (takes over screen on focus — mobile default), `bar` (persistent in nav — web default)
- **States**: `empty`, `typing`, `results`, `no-results`
- **Props**: placeholder text, cancel/clear action

## Dialog

- **Variants**: `alert` (title + message + 1-2 actions), `confirmation` (destructive action, requires explicit confirm), `sheet` (see Bottom Sheet)
- **States**: `default`
- **Radius token**: `radius.dialog`
- **Usage rule**: destructive actions always use `confirmation` variant, never `alert`

## Bottom Sheet (mobile) / Modal (web fallback)

- **Variants**: `content` (custom content, e.g. filter options), `action-list` (list of tappable actions — iOS calls this an action sheet)
- **Radius token**: `radius.bottom_sheet` (top corners only)
- **Platform note**: on iOS, `action-list` maps to native action sheet convention; on Android, to a modal bottom sheet — see platform-conventions

## Toast / Snackbar

- **Variants**: `success`, `error`, `info`
- **Behavior**: auto-dismiss (default 3s), optional single action button, never blocks input
- **Placement**: bottom of screen on mobile, top-right on web (unless platform convention overrides)

## Empty State

- **Props**: illustration/icon slot, headline, supporting text, optional primary action
- **Usage rule**: every list/collection screen must define an empty state — this is a common PM blind spot, the Starter Kit generates a default even if not explicitly requested

## Loading

- **Variants**: `skeleton` (content-shaped placeholder — default for lists/cards), `spinner` (default for full-screen or button-level loading), `progress-bar` (determinate, long operations only)
- **Usage rule**: prefer `skeleton` over `spinner` for anything that loads a known layout (feeds, lists, detail pages) — reduces perceived wait

---

## Numeric Typography (conditional component, finance/data products only)

Activated when intake detects a financial/market-data product type. See
`modules/preset-blender.md` → product-type extensions.

Current Price has **two size tiers**, because the same number reads very differently
embedded in a list/card/chat bubble versus as the sole focus of a screen. Picking one
size for both under- or over-sizes it in the other context (confirmed in the PM dry-run:
28px was right inline, but would read small as the hero number on a dedicated stock
detail screen). Both tiers are derived formulaically from the resolved type scale, not
hand-picked per preset:

All sizes below use the **platform-resolved** `typography.scale_px` from the manifest (see
`modules/preset-blender.md` Step 4b) — never the raw preset ratio. This matters most here:
an uncapped hero number is the single most common "why is this text gigantic" complaint on
mobile, so the Detail tier is the one the mobile cap in Step 4b protects.

- **Current Price — Inline** (used in cards, list rows, chat bubbles, anywhere the price
  shares space with other content): size = the resolved **H1** level (`scale_px.h1`)
- **Current Price — Detail** (used when the price is the primary subject of the whole
  screen, e.g. a stock detail page): size = `scale_px.numeric_current_price_detail`
  (`base * effective_ratio^5`, mobile-capped) — the largest text on screen, but on a phone
  still bounded so it never blows past a sensible size
- **Percentage / Delta**: tabular-nums, paired with `color.up` / `color.down`, includes
  directional icon. Sized to match whichever Current Price tier it's paired with (Body
  size next to Inline, H1 size next to Detail).
- **Market Data (table/list rows)**: tabular-nums, right-aligned, smaller than Current
  Price — always the Inline tier, Detail never applies to tabular rows
- **Rule**: all numeric typography uses tabular (monospaced-digit) figures so columns of
  numbers align — this is a correctness rule, not a style choice
