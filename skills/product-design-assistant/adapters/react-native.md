# Adapter: React Native

Reads `.design/design-manifest.json` and writes a typed theme module engineers import
directly — RN has no CSS cascade, so tokens must be plain JS/TS objects consumed via a
theme provider or direct import.

## Files written

### 1. `src/theme/tokens.ts` (fully generated — safe to overwrite each run)

```ts
// >>> product-design-assistant generated — edit .design/design-manifest.json instead
export const colors = {
  primary: { 50: '{ramp[0]}', 500: '{ramp[4]}', 900: '{ramp[8]}' },
  background: '{manifest.color.background}',
  surface: '{manifest.color.surface}',
  border: '{manifest.color.border}',
  divider: '{manifest.color.divider}',
  text: { primary: '{...}', secondary: '{...}', onPrimary: '{...}' },
  success: '{...}', warning: '{...}', error: '{...}', info: '{...}',
  // + up/down/chart/risk if finance extension active
} as const;

// fontSize values come straight from manifest.typography.scale_px (platform-resolved in
// preset-blender.md Step 4b) — do NOT recompute from scale_ratio in the adapter.
export const typography = {
  // fontWeight: RN takes a string enum — emit manifest.typography.weight as a string.
  // letterSpacing: RN has no em, the value is in POINTS. Convert per level:
  //   round(manifest.typography.letter_spacing_em[level] * scale_px[level], 2)
  // Do the arithmetic when generating this file; never leave an expression in the output.
  // lineHeight: RN takes POINTS, not a ratio — round(line_height[level] * scale_px[level]).
  // Unlike CSS there is no unitless form, so every level must be computed here; omitting
  // it makes RN fall back to the font's own metrics, which differ per platform.
  display: { fontSize: {scale_px.display}, fontWeight: '{weight.display}',
             letterSpacing: {letter_spacing_em.display * scale_px.display, rounded},
             lineHeight: {round(line_height.display * scale_px.display)} },
  h1: { fontSize: {scale_px.h1}, fontWeight: '{weight.h1}',
        letterSpacing: {letter_spacing_em.h1 * scale_px.h1, rounded},
        lineHeight: {round(line_height.h1 * scale_px.h1)} },
  // h2, h3, body, caption, button, label ...
  // numericCurrentPrice / numericPercentage / numericMarketData if finance extension active
  //   each includes fontVariant: ['tabular-nums'] — RN's equivalent of CSS tabular-nums
} as const;

// Family — RN cannot load a font from a URL. The font files must be linked into the
// native projects (react-native.config.js + `npx react-native-asset`, or manually via
// Info.plist / android/app/src/main/assets/fonts). Emit the names here AND state the
// linking step in component-guide.md; a fontFamily naming an unlinked font fails
// silently on iOS and crashes some Android builds.
//
// Android does NOT combine fontFamily with fontWeight: it needs the weight-specific
// PostScript name. So emit one entry per weight the manifest actually uses, and have
// components reference these instead of setting fontWeight on a custom family.
export const fonts = {
  regular: '{latin_face}-Regular', medium: '{latin_face}-Medium',
  semibold: '{latin_face}-SemiBold', bold: '{latin_face}-Bold',
} as const;
// CJK glyphs fall through to the OS face (PingFang TC / Noto Sans CJK) unless the CJK
// font is also linked — on RN there is no per-glyph fallback chain like CSS gives you,
// so a mixed-script string renders in whatever the single named family covers.

export const spacing = { 1: N, 2: N, 3: N, 4: N, 5: N, 6: N, 7: N, 8: N } as const; // from manifest.spacing.scale

export const radius = {
  button: N, card: N, input: N, dialog: N, bottomSheet: N,
} as const;

export const elevation = {
  // RN has no CSS box-shadow — use platform-specific shadow props
  1: { shadowColor: '#000', shadowOpacity: N, shadowRadius: N, elevation: N /* Android */ },
  2: { ... },
  3: { ... },
} as const;

export const icon = { strokeWidth: N, gridSize: N } as const; // pass to whatever icon library the project uses
// <<< product-design-assistant generated
```

### 2. `src/theme/ThemeProvider.tsx` (only created if it doesn't already exist — never
overwrite a PM's/engineer's existing provider)

A thin context provider exposing `tokens` above, so components use `useTheme()` instead
of importing the raw token file everywhere.

### 3. `.design/component-guide.md`

Same purpose as the Tailwind adapter's — one entry per component × variant × size, but
as RN `StyleSheet.create` recipes instead of Tailwind classes, e.g.:

```md
## Button — primary, md
StyleSheet: paddingHorizontal: spacing[6], paddingVertical: spacing[3],
borderRadius: radius.button, backgroundColor: colors.primary[500]
```

## Platform-conventions integration

Since one React Native codebase typically ships to both iOS and Android, this adapter
must apply **both** `platform-conventions/ios.md` and `platform-conventions/android.md`
using RN's `Platform.select()` pattern for anything that differs (tab bar behavior,
action sheet vs modal bottom sheet, dialog button alignment). Don't pick one platform's
convention and apply it to both.

```ts
const bottomSheetVariant = Platform.select({ ios: 'action-sheet', android: 'modal-sheet' });
```

## Numeric typography

If the finance extension is active, every numeric text component must set
`fontVariant: ['tabular-nums']` — flag this in the component guide, it's the single most
common thing engineers forget on RN and it breaks column alignment in price/data lists.

## Idempotency

Same marker convention as the Tailwind adapter — `tokens.ts` is fully regenerated each
run; `ThemeProvider.tsx` is only scaffolded once and never touched again automatically.
