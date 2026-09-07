# Validator: Contrast Check

Goal: guarantee every generated color pairing meets WCAG AA — this is the "quality
floor" that justifies using presets instead of pure live generation. Never skip this,
even on a small iteration tweak.

## A color is not one value

Every semantic color resolves to five roles, because the same hue has different
obligations depending on what it is doing. This is the whole design of this validator:
a single hex checked against a single threshold is what produced palettes that were
simultaneously over-corrected (chart lines darkened that never needed it) and
under-corrected (badge text that was never checked against its own badge).

| Role | Sits on | Threshold | Used by |
|------|---------|-----------|---------|
| `fill` | — | **none** | decorative area fills, chart area fills, brand marks — nothing that carries meaning on its own |
| `graphic` | `surface` | **3:1** (SC 1.4.11) | icons, chart lines, input borders, focus rings — anything needed to understand or operate the UI |
| `tint` | `surface` | none (it is a surface, not a mark) | badge / chip / row-highlight backgrounds |
| `text` | its own `tint` | **4.5:1** | the color used as text, anywhere |
| `action` | pair | **4.5:1** between `bg` and `fg` | the color used as a filled control |

Two consequences worth stating outright, because both were wrong before:

- **`text` is calibrated against `tint`, not against `surface`.** A badge's label sits on
  the badge, not on the page. Calibrating against white and then placing the result on a
  pale tint loses roughly half a point of ratio — enough to drop a passing 4.5 to a
  failing 4.0 across an entire palette. Because `tint` is darker than `surface`, a `text`
  that passes on `tint` passes on `surface` for free, so one value covers both.
- **`fill` has no threshold and therefore never changes.** Decorative fills and logos are
  exempt under SC 1.4.11. The brand color survives contrast validation intact; what
  validation adjusts is the *other four roles*.

## Order of correction — try the text first

When a pairing fails, correct in this order and stop at the first that passes:

1. **Change the foreground, not the color.** For an `action` pair, try `#FFFFFF`, then the
   preset's `text.primary` ink. A vivid brand color usually fails with white and passes
   easily with dark ink — friendly-playful's signature coral `#FF6B4A` is 2.82:1 against
   white and 5.44:1 against its own warm black. Dark text on a bright control is a normal,
   deliberate look, not a compromise.
2. **Darken the role, not the brand.** Walk the *role's* value down in lightness (HSL, so
   hue and saturation survive) until it passes. `fill` is never walked.
3. **Only then darken the fill**, and only for that one role's use as `action.bg`. Record
   which role changed, so `fill` stays available everywhere else.

Correcting in the old order — darken the color first — is what makes an entire palette go
muddy to fix a problem that only ever existed in one place. Reversing it costs nothing and
keeps the preset recognisable.

## Pairings to check

- `text.primary` and `text.secondary` on `background` and on `surface`
- every semantic color's `text` on its own `tint` **and** on `surface`
- every semantic color's `graphic` on `surface` **and** on `background`
- every semantic color's `action.fg` on its `action.bg`
- borders and dividers against the surface they separate (3:1 only where they carry
  meaning — a decorative hairline between rows does not, an input's border does)
- focus rings against both the control and the surface behind it
- if the finance extension is active: `up`/`down` `text` on `surface`, on `tint`, and on
  the alternating row background if the table has one

## Thresholds

- Text: **4.5:1** (WCAG AA, SC 1.4.3). The 3:1 large-text allowance exists but this
  validator does not use it — a size that qualifies today stops qualifying the moment the
  type scale is re-resolved, and the saving is not worth a rule that silently expires.
- UI components and meaningful graphics: **3:1** (SC 1.4.11). Not rounded — 2.999:1 fails.
- Disabled states and purely decorative graphics: exempt.

## Contrast ratio formula

```
L = 0.2126*R + 0.7152*G + 0.0722*B   (R,G,B linearized from sRGB)
contrast = (L_lighter + 0.05) / (L_darker + 0.05)
```

## On failure

1. Do not ask the PM to fix it — this is invisible infrastructure, not a design decision
   they should be handed.
2. Apply the correction order above.
3. If a role cannot pass even at the darkest end of its own hue (rare — an extreme brand
   color, e.g. a pale yellow forced as primary), fall back to the preset's original value
   for that one role and log it in plain language:
   > "品牌黃色配白字看不清楚,按鈕文字改用深色,顏色本身沒有動。"
4. Every correction gets one line in `manifest.validation_log`, naming the role, both
   values, both ratios. This is what makes "AA validated" auditable later rather than
   asserted.

## Where this runs

Called from `preset-blender.md` Step 6 (fresh build) and from the iteration flow whenever
a color-related field changes. Never run adapters against a manifest that has not passed.
