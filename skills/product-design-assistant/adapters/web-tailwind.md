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

  /* Typography */
  --font-size-display: {calc from base_size_px * scale_ratio^4};
  --font-size-h1: ...;
  /* ... h2, h3, body, caption, button, label — all derived the same way */

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
      display: 'var(--font-size-display)',
      h1: 'var(--font-size-h1)',
      // ...
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
text-white font-button disabled:opacity-40`
```

One recipe per component × variant × size combination that the manifest defines.

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
