# Validator: Contrast Check

Goal: guarantee every generated color pairing meets WCAG AA — this is the "quality
floor" that justifies using presets instead of pure live generation. Never skip this,
even on a small iteration tweak.

## Pairings to check

For every text/icon color against every background/surface it's meant to sit on:

- `text.primary` on `background`
- `text.primary` on `surface`
- `text.secondary` on `background` / `surface`
- `text.on_primary` (usually white) on `color.primary` (button labels)
- each semantic color (`success`/`warning`/`error`/`info`) on `background` and as a
  filled button background with `text.on_primary` on top
- if finance extension active: `up`/`down` on `background`, and on `surface` (table rows)

## Thresholds

- Body text / icons: contrast ratio ≥ 4.5:1 (WCAG AA normal text)
- Large text (H1–H3, Display, Button labels ≥ 18px bold or ≥ 24px): ≥ 3:1
- Non-text UI (borders, dividers, focus rings): ≥ 3:1 against adjacent color

## Contrast ratio formula

Standard WCAG relative luminance formula:

```
L = 0.2126*R + 0.7152*G + 0.0722*B   (R,G,B linearized from sRGB)
contrast = (L_lighter + 0.05) / (L_darker + 0.05)
```

## On failure

1. Do not ask the PM to fix it — this is invisible infrastructure work, not a design
   decision they should be bothered with.
2. Walk the failing color's ramp (the same hue, adjusted lightness) toward darker (on
   light backgrounds) or lighter (on dark backgrounds) in the smallest steps available
   until the pairing passes.
3. If the ramp itself doesn't have enough range to pass (rare — only happens with an
   extreme brand color choice, e.g. a very light yellow forced as primary), fall back to
   the preset's `default_hex` for that specific token only, and note it in
   `manifest.validation_log` with a one-sentence plain-language reason:
   > "品牌黃色在白底文字上對比不足,按鈕文字改用深色文字而非白色,確保可讀性。"
4. Every correction — automatic or fallback — gets one line in `validation_log`. This is
   what makes the "AA validated" claim auditable later instead of just asserted.

## Where this runs

Called from `preset-blender.md` Step 6 (fresh build) and from the iteration flow
whenever a color-related field changes. Never run adapters against a manifest that
hasn't passed this check.
