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
  "typography": { "...": "resolved scale + numeric typography if active" },
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
   - anything ambiguous → ask ONE clarifying multiple-choice question, don't guess silently
3. Re-run Step 6 (contrast validation) on the changed tokens only.
4. Bump `manifest_version`, rewrite the manifest, re-run the relevant adapter to update
   the actual project files.
5. Tell the PM what changed in one plain sentence — don't re-explain the whole system
   every time.

Never regenerate the entire manifest from scratch on a small tweak — that's how
"team-shared standard" consistency breaks down over time.
