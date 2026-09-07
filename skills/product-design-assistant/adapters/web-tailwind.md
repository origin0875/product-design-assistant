# Adapter: Web + Tailwind

Reads `.design/design-manifest.json` and writes real, working files into the PM's
project. Never invent values here — every value comes from the manifest; this file only
decides *where it goes and in what syntax*.

## Files written

### 1. `.design/tokens.css`

CSS custom properties, generated from the manifest — the portable, framework-agnostic
layer underneath Tailwind:

```css
:root {
  /* Color */
  --color-primary-50: {manifest.color.primary_ramp[0]};
  --color-primary-500: {manifest.color.primary_ramp[4]};
  --color-primary-900: {manifest.color.primary_ramp[8]};
  --color-background: {manifest.color.background};
  --color-surface: {manifest.color.surface};
  --color-border: {manifest.color.border};
  --color-divider: {manifest.color.divider};
  --color-text-primary: {manifest.color.text_primary};
  --color-text-secondary: {manifest.color.text_secondary};
  --color-success: {manifest.color.success};
  --color-warning: {manifest.color.warning};
  --color-error: {manifest.color.error};
  --color-info: {manifest.color.info};
  /* + up/down/chart/risk if finance extension active */

  /* Typography — read verbatim from manifest.typography.scale_px (already resolved in
     preset-blender.md Step 4b). Never recompute from effective_ratio here. */
  --font-size-display: {manifest.typography.scale_px.display}px;
  --font-size-h1: {manifest.typography.scale_px.h1}px;
  /* ... h2, h3, body, caption, button, label — each straight from scale_px */

  /* Weight — from manifest.typography.weight (resolved in preset-blender.md Step 4c).
     Emit all eight levels even where several share a value: the preset's ladder is what
     separates 專業穩重 from 簡約優雅, and collapsing it here erases that difference. */
  --font-weight-display: {manifest.typography.weight.display};
  --font-weight-h1: {manifest.typography.weight.h1};
  /* ... h2, h3, body, caption, button, label — each straight from weight */

  /* Tracking — from manifest.typography.letter_spacing_em (Step 4c). Stored in em, used
     in em: it is size-relative, so it stays correct if the scale is re-resolved later. */
  --letter-spacing-display: {manifest.typography.letter_spacing_em.display}em;
  --letter-spacing-h1: {manifest.typography.letter_spacing_em.h1}em;
  /* ... h2, h3, body, caption, button, label — each straight from letter_spacing_em */

  /* Line-height — from manifest.typography.line_height (Step 4d). Unitless, never px:
     a unitless ratio is inherited as a ratio, so nested text at a different size still
     gets correct leading. The one exception is body when Step 4d snapped it to the 4px
     grid — emit that as px so the snap actually survives. */
  --line-height-display: {manifest.typography.line_height.display};
  --line-height-body: {manifest.typography.line_height_px.body}px;  /* or the ratio if unsnapped */
  /* ... h1, h2, h3, caption, button, label — each straight from line_height */

  /* Family — from manifest.typography.font_family (Step 4e). Emit the stack verbatim,
     Latin face first: the browser resolves per glyph, so Latin and digits take the Latin
     face and Chinese falls through to the CJK face. Do not reorder. */
  --font-family-body: {manifest.typography.font_family.body};
  --font-family-display: {manifest.typography.font_family.display};

  /* Spacing */
  --space-1: {scale[0]}px; --space-2: {scale[1]}px; /* ... through scale[7] */

  /* Radius */
  --radius-button: {resolved}px;
  --radius-card: {resolved}px;
  --radius-input: {resolved}px;
  --radius-dialog: {resolved}px;
  --radius-bottom-sheet: {resolved}px;

  /* Elevation */
  --shadow-1: 0 1px 2px rgba(0,0,0,{opacity_range[0]});
  --shadow-2: 0 4px 8px rgba(0,0,0,{mid});
  --shadow-3: 0 12px 24px rgba(0,0,0,{opacity_range[1]});
}
```

### 1b. The webfont link (only when `manifest.typography.font_source == "google"`)

Add to the app's document head, requesting exactly `manifest.typography.webfont_weights`
— not a blanket 100..900, which on a CJK family is a large amount of weight the page
never uses:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family={latin}:wght@{latin_weights}&family={cjk}:wght@{cjk_weights}&display=swap">
```

`display=swap` is not optional: a CJK family is fetched as many `unicode-range` slices, and
without it the page renders invisible text until they land.

If `font_source` is `"system"`, write no link at all and say so in the component guide —
a stack that names a face the project never loads is the most common way a design system
looks correct in the manifest and wrong in the browser.

### 2. `tailwind.config.js` (extend, never replace)

If the file exists, extend `theme.extend` — do not overwrite the PM's existing config
wholesale. Map every CSS variable above into Tailwind's theme so engineers use
`bg-primary-500`, `rounded-card`, `shadow-2`, `text-h1` etc. instead of raw hex/px:

```js
theme: {
  extend: {
    colors: {
      primary: { 50: 'var(--color-primary-50)', 500: 'var(--color-primary-500)', 900: 'var(--color-primary-900)' },
      background: 'var(--color-background)',
      surface: 'var(--color-surface)',
      // ...
    },
    fontSize: {
      // Keep these scalar (not Tailwind's [size, {...}] tuple form) so size, weight and
      // tracking stay three composable classes: text-h1 font-h1 tracking-h1.
      display: 'var(--font-size-display)',
      h1: 'var(--font-size-h1)',
      // ...
    },
    fontWeight: {
      display: 'var(--font-weight-display)',
      h1: 'var(--font-weight-h1)',
      // ... h2, h3, body, caption, button, label
      // Without this block `font-button` in the component recipes below is not a real
      // Tailwind class and silently renders at the browser default weight.
    },
    letterSpacing: {
      display: 'var(--letter-spacing-display)',
      h1: 'var(--letter-spacing-h1)',
      // ... h2, h3, body, caption, button, label
    },
    lineHeight: {
      display: 'var(--line-height-display)',
      body: 'var(--line-height-body)',
      // ... h1, h2, h3, caption, button, label
    },
    fontFamily: {
      // Tailwind wants arrays; split the resolved stack on commas rather than
      // re-authoring it, so the Latin-first order from Step 4e survives.
      body: ['var(--font-family-body)'],
      display: ['var(--font-family-display)'],
    },
    spacing: { /* map scale to Tailwind's spacing keys */ },
    borderRadius: {
      button: 'var(--radius-button)',
      card: 'var(--radius-card)',
      // ...
    },
    boxShadow: { 1: 'var(--shadow-1)', 2: 'var(--shadow-2)', 3: 'var(--shadow-3)' },
  },
},
```

### 3. `.design/component-guide.md`

Human-readable (for engineers, not PM) mapping of each component in
`component-structure.md` to Tailwind class recipes, e.g.:

```md
## Button — primary, md
`inline-flex items-center justify-center px-6 py-3 rounded-button bg-primary-500
text-white text-button font-button tracking-button leading-button disabled:opacity-40`
```

One recipe per component × variant × size combination that the manifest defines.

## The type set — four classes, always together

Every text-bearing element in the component guide must carry all four of
`text-{level}`, `font-{level}`, `tracking-{level}` and `leading-{level}` — never a size
class on its own. A lone `text-h1` inherits whatever weight, tracking and leading the
surrounding element happened to set, which is exactly the drift
`modules/typography-audit.md` later has to clean up. State this rule at the top of the
generated `component-guide.md`, not just in the recipes, so engineers writing new screens
follow it too.

Buttons are the one place `leading-button` matters more than it looks: without it the
button inherits body leading (up to 28px on a CJK build) and the control grows ~12px
taller than the spec, which then desynchronises it from input height beside it.

## Numeric typography (if finance extension active)

Add a `.tabular-nums` utility usage note and a dedicated `text-market-data` /
`text-current-price` scale entry using `font-variant-numeric: tabular-nums` — flag this
explicitly in the component guide since it's easy for engineers to forget and get
misaligned number columns.

## Idempotency

Re-running this adapter (after a manifest iteration) overwrites `.design/tokens.css` and
`.design/component-guide.md` fully (they're fully generated, safe to replace), but only
patches the `theme.extend` block inside `tailwind.config.js` via a clearly marked region:

```js
// >>> product-design-assistant generated — do not edit below by hand, edit .design/design-manifest.json instead
...
// <<< product-design-assistant generated
```

Only replace content between these markers on regeneration.
