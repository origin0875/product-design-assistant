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

  /* Typography — read verbatim from manifest.typography.scale_px (already platform-resolved
     in preset-blender.md Step 4b). Never recompute from scale_ratio here. */
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
text-white text-button font-button tracking-button disabled:opacity-40`
```

One recipe per component × variant × size combination that the manifest defines.

## The type trio

Every text-bearing element in the component guide must carry all three of
`text-{level}`, `font-{level}` and `tracking-{level}` — never a size class on its own.
A lone `text-h1` inherits whatever weight and tracking the surrounding element happened
to set, which is exactly the drift `modules/typography-audit.md` later has to clean up.
State this rule at the top of the generated `component-guide.md`, not just in the
recipes, so engineers writing new screens follow it too.

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
