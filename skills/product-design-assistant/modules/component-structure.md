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

## Control sizing — applies to every interactive component

Two different numbers, and conflating them is the usual mistake:

- **Visual height** is how tall the control looks.
- **Hit target** is the area that responds to a finger or pointer. It may be larger than
  the visual box, and on touch platforms it usually must be.

**Visual height by size:**

| Size | Height | Used by |
|------|--------|---------|
| `sm` | 32 | dense toolbars, table-row actions, chips |
| `md` | 40 | the default for Button, Input, Select |
| `lg` | 48 | primary actions on mobile, full-width CTAs |

**Padding is derived from height, never chosen independently.** Vertical padding =
`(height − line_box) / 2`, where `line_box` is that level's resolved size × line-height
(Steps 4b/4d). Authoring padding directly is exactly how a control ends up 37px tall: each
padding value looked like a reasonable number from the spacing scale, and nobody added them
up. Horizontal padding keeps the existing rule — at least 1.5× the *derived* vertical
padding.

**Hit target minimum** is resolved per platform (`platform-conventions/*.md`); web has no
convention file, so its floor lives in the web adapter. Where the visual height sits below
that floor — a 40px `md` button on a phone, any icon-only control — **extend the interactive
area without changing the visual box**: a transparent inset overlay on web, `hitSlop` on
React Native. Do not inflate the control to reach the floor. Material's own filled button is
40dp tall inside a 48dp touch target for this reason; enlarging the visible box instead is
what makes a design look like it was drawn for a tablet.

The smallest interactive thing on a screen is where this always fails — an icon-only close,
a chip's remove ✕, a sort caret. Those have no label to give them height, so their visual box
is just the icon and the hit area has to be added deliberately. Check them explicitly; they
are the ones that get shipped at 14px.

## Button

- **Sizes**: `sm`, `md`, `lg` (heights from **Control sizing** above)
- **Variants**: `primary` (solid), `secondary` (outline or tonal — preset decides), `tertiary` (text-only), `destructive`
- **States**: `default`, `hover` (web only), `pressed`, `disabled`, `loading`
- **Props**: label, optional leading/trailing icon, full-width flag
- **Padding**: vertical derived from the size's height (see **Control sizing**);
  horizontal ≥ 1.5× that derived vertical padding
- **Radius token**: `radius.button`
- **Usage rule**: exactly one `primary` button visible per screen/section — never two competing primaries

## Input

- **Sizes**: `md`, `lg` (no `sm` — an input at 32px cannot carry a comfortable target)
- **Variants**: `text`, `number` (uses numeric typography if preset defines it), `search` (see Search), `textarea`
- **States**: `default`, `focused`, `filled`, `error`, `disabled`
- **Props**: label (always visible, never placeholder-only), helper text, error text, leading/trailing icon
- **Radius token**: `radius.input`

## Directional Value (finance extension only)

Any number whose meaning includes a direction — price change, percentage change, P&L,
net flow — is a single component, not a coloured `<span>`.

- **Props**: value, direction (`up` / `down` / `flat`), and the indicator style resolved in
  `preset-blender.md` Step 3b
- **Required**: the indicator (a `+`/`-` sign, ▲/▼, or an arrow) renders **always**, not
  only when color is unavailable. Color is reinforcement; the glyph is the message.
  This is WCAG SC 1.4.1, Level A — and red/green is precisely the pair the most common
  color-vision deficiencies cannot tell apart, so the rule earns its place here on merit,
  not only on compliance.
- **Flat**: a zero change takes the neutral color and no directional glyph — never the
  "up" treatment with a `+0.00`
- **Screen readers**: the accessible name says the direction in words ("上漲 1.40%"), since
  ▲ alone announces as a shape or not at all
- **Usage rule**: never place a directional value inside a cell whose background already
  carries the same hue for another meaning (a red "up" figure on a red risk badge)

## Text Size Control

- **Variants**: `segmented` (three labelled options side by side — the default) or `list`
  (a settings row per option, for a dense settings screen)
- **Options**: 小 / 中 / 大, exactly three, always labelled with words rather than only
  glyph sizes — an A-A-A control tells a user nothing about which one they are on
- **States per option**: `selected`, `unselected`
- **Placement**: wherever the product keeps display settings; never buried more than two
  taps deep
- **Behaviour**: applies immediately and persists per user. Do not require a restart, and
  do not scope it to one screen.
- **Usage rule**: the control must render its own preview text at the size being offered,
  so the choice is visible before it is made

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
