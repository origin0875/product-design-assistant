# Module: Preset Blender

Goal: turn `intake_answers` (from `intake.md`) + a chosen preset (from `presets/*.yaml`)
into one concrete `design-manifest.json` — the single source of truth every adapter,
the future Screen Generator, and Design Review will read from.

## Step 1 — Select base preset

Use `intake_answers.tone_choice` to load the matching file from `presets/`. If the PM
picked "我不確定", apply the inference rule already defined in `intake.md` Q5.

## Step 2 — Blend in brand color (if provided)

If `intake_answers.brand_color.provided` is true:

1. Resolve the description/value to a concrete hex (e.g. "深藍色" → a reasonable navy
   hex; if the PM gave an exact hex/name, use it directly).
2. Convert to HSL.
3. **Keep the preset's structure, replace only the hue**: the preset's neutral scale,
   lightness ramp step count, and saturation profile stay as calibrated — only the hue
   angle changes to match the brand color. This is the "hybrid" rule agreed on: presets
   supply the vetted structure, brand color is the one live-computed input.
4. Regenerate the primary color ramp (e.g. 9 steps from `primary-50` to `primary-900`)
   using the preset's existing lightness/saturation steps with the new hue.

If not provided, use `preset.color.primary.default_hex` as-is and derive the ramp from
the preset's default structure.

## Step 3 — Activate product-type extensions

For each extension block in the preset (currently only `finance` exists), check if
`intake_answers.product_type == "finance"` OR `intake_answers.product_name`/description
matches any of the extension's `trigger_keywords`. If matched:

- Merge the extension's `color` block into the manifest's color tokens (adds `up`,
  `down`, `chart_line`, `chart_grid`, `risk_high/medium/low`)
- Set `typography.numeric_typography` to `required` and include the Numeric Typography
  spec from `modules/component-structure.md`

Extensions are additive only — they never remove or override base tokens.

## Step 4 — Apply density override

If `intake_answers.density_override` differs from `preset.component_bias.density`,
override `spacing.density` and adjust the spacing scale multiplier accordingly (dense →
tighten the scale ~15%, sparse → loosen ~15%; standard → use preset default unchanged).

## Step 4b — Resolve the type scale (platform-aware)

The preset's `scale_ratio` is calibrated for an **editorial / web** reading width. Applied
verbatim to a phone (~390pt wide), a dramatic ratio like minimal-elegant's 1.333 pushes the
top of the scale to sizes that read as "obviously too big" (e.g. a 67px hero number, a 50px
screen title). So the ratio must be resolved against the target platform, not copied raw.

**1. Pick the effective ratio.**

- If `intake_answers.platforms` contains **only** `web` → `effective_ratio = preset.scale_ratio` (unchanged).
- If it contains **any** mobile platform (`ios` / `android`) → this is a mobile-first build
  (one React Native codebase ships both), so **`effective_ratio = min(preset.scale_ratio, 1.25)`**.
  This keeps friendly-playful (1.25) and professional (1.20) intact, and only tames the most
  aggressive editorial ratio (minimal-elegant 1.333 → 1.25) where it actually hurts on a phone.

**2. Derive every level as a *consecutive* power of the effective ratio** — no gaps. A gap in
the scale is what forces engineers to hand-pick an off-scale size, which is exactly the
inconsistency this Skill exists to prevent:

| Token   | Formula                        |
|---------|--------------------------------|
| caption | `round(base / ratio)`   (^-1)  |
| body    | `base`                   (^0)  |
| button  | `base`                   (^0)  |
| label   | `round(base / ratio)`   (^-1)  |
| h3      | `round(base * ratio^1)`        |
| h2      | `round(base * ratio^2)`        |
| h1      | `round(base * ratio^3)`        |
| display | `round(base * ratio^4)`        |

The **only** intentionally non-consecutive size is the finance hero number below — every
general text level is a neighbour of the next, so there is never a "nothing fits here" hole.

**3. Cap the extremes on mobile** (safety net on top of the ratio cap):

- `display` (mobile) = `min(display, 40)`
- `numeric_current_price_detail` (mobile) = `min(round(base * ratio^5), 52)`

On web, no cap — editorial hierarchy is the point there.

**4. Numeric typography (finance), if active** — resolved from the *same* effective ratio so
mobile and web stay internally consistent:

- `numeric_current_price_inline` = the resolved **h1** size
- `numeric_current_price_detail` = `base * ratio^5`, then mobile-capped per step 3
- `numeric_market_data` = the resolved **body** size, tabular

Write the fully-resolved pixel values into `typography.scale_px` in the manifest (see Step 7).
Adapters must read those resolved values verbatim — they never recompute from the ratio, so
the platform decision made here is the single place it lives.

## Step 4c — Resolve weight + letter-spacing

`typography.weight` and `typography.letter_spacing_bias` are the two preset fields that
were previously copied toward the manifest without ever becoming usable numbers — the web
adapter had no weight or tracking output at all, so on web those presets' distinctions
silently evaporated. Resolve both here, for the same reason as Step 4b: adapters read
verbatim, they never interpret.

**1. Weight.** Copy `preset.typography.weight` into `manifest.typography.weight` for all
eight levels (display, h1, h2, h3, body, caption, button, label) as integers. No platform
adjustment — the weight ladder *is* the differentiator between presets (professional steps
700→600→400; friendly-playful raises the whole ladder and puts button at 700;
minimal-elegant deliberately flattens to 500/400 and carries hierarchy with size and
whitespace instead). Never substitute a weight the preset didn't specify, and never
collapse the eight levels into "bold / regular".

**2. Letter-spacing.** Map `preset.typography.letter_spacing_bias` to per-level `em`
values. A single global number is always wrong here: optical need runs in opposite
directions at the two ends of the scale — large type needs tightening, small type needs
opening up.

| Token   | tight   | normal  | wide    |
|---------|---------|---------|---------|
| display | -0.02em | -0.01em | 0       |
| h1      | -0.02em | -0.01em | 0       |
| h2      | -0.01em | 0       | +0.01em |
| h3      | -0.01em | 0       | +0.01em |
| body    | 0       | 0       | +0.01em |
| button  | 0       | 0       | +0.02em |
| label   | 0       | +0.01em | +0.02em |
| caption | 0       | +0.01em | +0.02em |

**CJK guard — not optional.** Note that negative tracking appears only at h3 and above,
never at body/caption/label/button. Chinese glyphs fill the full em square and have no
side bearing to borrow from, so negative tracking on CJK running text pushes strokes into
each other and reads as a rendering fault rather than as refinement. Additionally, if the
product's primary language is CJK (the PM's product name/description is in Chinese, or
they said so in intake), clamp display/h1/h2/h3 to `max(value, -0.01em)` and log the clamp
in `validation_log`.

Write both into the manifest as `typography.weight` and `typography.letter_spacing_em`
(Step 7). Units: `em` is stored because it is size-relative and therefore survives any
later scale change; the CSS adapters consume it as-is, and the React Native adapter
converts to points (RN has no `em`) — see `adapters/react-native.md`.

## Step 5 — Resolve platform + component structure

- Pull the canonical component list/variants/states from `modules/component-structure.md`
  (unchanged structure, just referenced).
- For each platform in `intake_answers.platforms`, note which
  `platform-conventions/{ios,android}.md` file(s) apply. Web has no platform-convention
  file — its conventions live directly in the adapter.

## Step 6 — Validate contrast

Run every text/background and icon/background pairing in the resolved manifest through
`validators/contrast-check.md`. Any failing pair is replaced with the nearest passing
step from the same ramp (never hand-waved) and logged in `manifest.validation_log`.

## Step 7 — Write the manifest

Write to `.design/design-manifest.json` in the PM's project. Structure:

```json
{
  "meta": {
    "product_name": "...",
    "product_type": "finance",
    "platforms": ["ios", "android"],
    "primary_users": "...",
    "preset_base": "professional-trustworthy",
    "manifest_version": 1
  },
  "principle": { "...": "from preset, verbatim" },
  "color": { "primary_ramp": ["..."], "...": "resolved tokens" },
  "typography": {
    "scale_ratio": "preset value (kept for reference)",
    "effective_ratio": "platform-resolved ratio from Step 4b",
    "scale_px": { "caption": 0, "body": 0, "h3": 0, "h2": 0, "h1": 0, "display": 0,
                  "numeric_current_price_inline": 0, "numeric_current_price_detail": 0, "numeric_market_data": 0 },
    "weight": { "display": 0, "h1": 0, "h2": 0, "h3": 0,
                 "body": 0, "caption": 0, "button": 0, "label": 0 },
    "letter_spacing_bias": "preset value (kept for reference)",
    "letter_spacing_em": { "display": 0, "h1": 0, "h2": 0, "h3": 0,
                           "body": 0, "caption": 0, "button": 0, "label": 0 }
  },
  "spacing": { "...": "resolved scale" },
  "radius": { "...": "resolved values" },
  "elevation": { "...": "resolved" },
  "icon": { "...": "resolved" },
  "layout": { "...": "resolved" },
  "components": { "...": "reference to component-structure.md + resolved styling" },
  "validation_log": ["e.g. contrast auto-corrected: error color darkened from #E5484D to #D6373C for AA on white"]
}
```

`manifest_version` increments on every regeneration — lets adapters/Design Review (later
phase) know they're reading stale vs current output.

## Iteration flow (manifest already exists)

When a PM says something like "品牌色改藍色" or "按鈕圓角改大一點" in a follow-up
message:

1. Read the existing `.design/design-manifest.json`.
2. Map the request to the smallest set of manifest fields it touches (a small lookup,
   not a full re-interview):
   - color mentions → re-run Step 2 only, keep everything else
   - "圓角/radius/更圓/更方" → bump/reduce the relevant `radius.scale_px` step(s)
   - "太擠/太鬆/density" → re-run Step 4 only
   - "字太細/太粗/標題不夠重" → adjust `typography.weight` for the named levels only,
     via Step 4c (do not swap the whole preset just to change weight)
   - "字距太擠/太開" → re-run Step 4c's letter-spacing table only, keeping the CJK guard
   - anything ambiguous → ask ONE clarifying multiple-choice question, don't guess silently
3. Re-run Step 6 (contrast validation) on the changed tokens only.
4. Bump `manifest_version`, rewrite the manifest, re-run the relevant adapter to update
   the actual project files.
5. Tell the PM what changed in one plain sentence — don't re-explain the whole system
   every time.

Never regenerate the entire manifest from scratch on a small tweak — that's how
"team-shared standard" consistency breaks down over time.
