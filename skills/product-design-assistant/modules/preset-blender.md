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

## Step 4b — Resolve the type scale

The type scale does **not** come from the preset. Which sizes exist, and how far apart they
sit, is information design: it encodes what matters most on the screen, and that ordering
does not change because the product's tone changed. Tone lives in weight, tracking,
leading, typeface and color — all of which this module still takes from the preset.

The practical reason matters as much as the principle. Ratio is the one typography value
whose blast radius is the whole layout: change it and every screen's line count, column
width and wrap points move. A PM switching tone expects a different feel, not a reflow of
every page they have already built.

**1. Pick the base ratio from information density.** `intake_answers.density_override`
already asks exactly this question in PM language ("很多數據/列表,使用者會頻繁掃描比對" /
"適中" / "很簡潔,一次只看少量內容") — Q7 in `intake.md`. Nothing new is asked.

| `density_override` | base ratio | why |
|--------------------|-----------|-----|
| `dense`    | 1.200 | scanning and comparing; small steps keep more rows in view and keep the jump between a label and its value from becoming a leap |
| `standard` | 1.250 | the general case |
| `sparse`   | 1.333 | reading one thing at a time; an editorial hierarchy is the point |

If the PM did not answer Q7, derive it from `product_type`: finance / B2B後台 / 工具 →
`dense`; 內容媒體閱讀 → `sparse`; everything else → `standard`.

**2. Apply the product-type floor.** A data product stays tight even when the PM asked for
a roomy feel — dramatic size steps hurt scanning regardless of how much whitespace sits
around them. If `product_type` is finance, B2B後台 or 工具, then
`base_ratio = min(base_ratio, 1.250)`. Whitespace is still free to grow: that is Step 4's
job, and it is the right lever for "看起來太擠" in a dense product.

**3. Apply the mobile cap.** If `intake_answers.platforms` contains any mobile platform
(`ios` / `android`), this is a mobile-first build — one React Native codebase ships both —
so `effective_ratio = min(base_ratio, 1.25)`. On a ~390pt phone a 1.333 scale pushes the
top of the ladder to sizes that read as obviously too big (a 67px hero number, a 50px
screen title). Web-only keeps the base ratio as-is.

**4. Derive every level as a *consecutive* power of the effective ratio** — no gaps. A gap in
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

**5. Cap the extremes on mobile** (safety net on top of the ratio cap):

- `display` (mobile) = `min(display, 40)`
- `numeric_current_price_detail` (mobile) = `min(round(base * ratio^5), 52)`

On web, no cap — editorial hierarchy is the point there.

**6. Numeric typography (finance), if active** — resolved from the *same* effective ratio so
mobile and web stay internally consistent:

- `numeric_current_price_inline` = the resolved **h1** size
- `numeric_current_price_detail` = `base * ratio^5`, then mobile-capped per step 3
- `numeric_market_data` = the resolved **body** size, tabular

Write the fully-resolved pixel values into `typography.scale_px` in the manifest (see Step 7).
Adapters must read those resolved values verbatim — they never recompute from the ratio, so
every decision made here lives in exactly one place.

Two presets on the same product therefore produce the **same** sizes and differ everywhere
else. That is the intended result, not a bug to be corrected by reintroducing a per-preset
ratio: it is what makes tone a safe thing for a PM to change their mind about.

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

**1b. Mobile CJK weight ceiling.** If `intake_answers.platforms` contains any mobile
platform **and** the product's primary language is CJK, clamp every level to **600**, and
use only the three steps 400 / 500 / 600.

The reason is that the ladder above cannot be rendered. The iOS system Chinese face,
PingFang TC, ships six weights — 極細 / 纖細 / 細 / 標準 / 中黑 / 中粗 — topping out at
中粗 (Semibold, 600). There is no 700. Asking for 700 gets Semibold or a synthetic bold,
while the Latin face beside it (SF) really does have 700 — so a mixed string like
「台積電 1,085」 renders the Chinese and the digits at visibly different weights. That is
why the clamp applies to **both scripts at that level**, not only to the CJK one: within-line
consistency is the thing being protected, and a level that reads 700 for half its characters
is worse than a level that is 600 throughout.

Android's bundled CJK face has a different weight set again, and may not carry 600 —
verify 600 on an Android device; where it synthesises, drop that level to 500 and log it.
A project that bundles its own CJK font (see Step 4e) escapes all of this and keeps the
preset's full ladder; note that option in `component-guide.md` when the clamp costs the
design something.

What the clamp costs is worth stating plainly to the PM: on professional-trustworthy it
collapses display/h1 (700) and h2/h3 (600) into a single weight, so on mobile the heading
hierarchy rests on size alone. That is the same choice Material 3's baseline scale makes —
its display, headline and title-large roles are all weight 400 — so it is a defensible
place to land, not a degradation. Log every clamped level in `validation_log`.

minimal-elegant needs no clamp: its ladder is already 500/400 throughout.

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

## Step 4d — Resolve line-height

Line-height is the single largest whitespace decision in the whole kit — a 16px body at
1.5 versus 1.8 changes the height of every screen — so it is calibrated per preset
(`typography.line_height_bias`), not improvised per project.

**1. Base ratios by bias.** Unitless, so they survive any later scale change. The ratio
falls as size rises: a 51px display line needs far less proportional leading than a 16px
paragraph.

| Token   | normal | relaxed | airy |
|---------|--------|---------|------|
| display | 1.15   | 1.15    | 1.20 |
| h1      | 1.20   | 1.25    | 1.25 |
| h2      | 1.25   | 1.30    | 1.30 |
| h3      | 1.35   | 1.40    | 1.40 |
| body    | 1.50   | 1.55    | 1.60 |
| label   | 1.40   | 1.45    | 1.45 |
| caption | 1.45   | 1.50    | 1.50 |
| button  | 1.20   | 1.20    | 1.20 |

`button` is fixed at 1.20 across all biases on purpose: button height comes from padding,
not from leading, so the ratio only needs to be large enough not to clip descenders.
Do not set it to 1.0 — that clips in several common CJK and geometric Latin faces.

**2. CJK adjustment.** If the product's primary language is CJK (same signal as the Step 4c
guard), add: **body `+0.20`**, label/caption `+0.15`, display/h1/h2/h3 `+0.05`, button `0`.
Chinese glyphs fill the em square edge to edge and have no ascender/descender gap to act as
visual leading, so Latin-calibrated line-height reads as a solid block of text. A
professional-trustworthy body therefore resolves to 1.50 for Latin and 1.70 for CJK.

**3. Grid snap — body only, and only when it is nearly on grid.** Compute
`body_line_px = body_size × ratio`. If it falls within **1px** of a multiple of 4, snap it
to that multiple and store the recomputed ratio; otherwise keep the ratio as-is. Body is
the level every other rhythm stacks against, so it is worth aligning — but only when the
correction is imperceptible. Beyond 1px, the preset's intent outranks the grid, because
snapping a 25.6px line box down to 24px would erase the exact difference between `airy`
and `normal` that the PM chose the preset for.

Note the one place this collapses: professional CJK (27.2px) and minimal-elegant CJK
(28.8px) both snap to 28px. That is a sub-pixel-per-line difference at that point and not
worth protecting — say nothing to the PM about it, but do not "fix" it by widening the
snap window either.

Write into the manifest as `typography.line_height` (the resolved ratios) and
`typography.line_height_px.body` (the snapped body line box, if snapping applied).

## Step 4e — Resolve font family

The preset names *candidate* faces; this step turns them into one stack per role, and
decides whether a webfont is loaded at all. Two mechanics have to be right or the stack
is decorative:

**1. Script order is the mechanism, not a preference.** Put the Latin face **before** the
CJK face in a single `font-family` list. The browser resolves per glyph, so Latin letters
and digits take the Latin face and Chinese characters fall through to the CJK face —
that is how one stack serves both scripts. Reversing the order silently hands Latin
digits to the CJK face, whose figures are usually full-width and differently
proportioned; in a finance product that also destroys column alignment against every
other number on the page. Never emit a stack with the CJK face first.

Always terminate the stack with the preset's `system_only` list plus a generic
(`sans-serif` / `serif`). A stack with no reachable last resort renders in the browser
default, which is the one outcome nobody chose.

**2. Weight availability, per script.** The preset ladders demand 500/600/700/800, and
CJK system faces do not all have them: PingFang TC covers Regular/Medium/Semibold, but
Microsoft JhengHei ships only Regular and Bold. Asking for 600 there produces synthetic
bold — smeared strokes, which on dense Chinese glyphs reads as a rendering fault. So:

- When a CJK **webfont** is loaded, request the weights the preset actually uses.
  Google Fonts delivers CJK families as many small `unicode-range` slices, so a page
  downloads only the slices its glyphs need and multiple weights stay affordable.
- When the build is **system-only** (no webfont — offline, an internal network that
  blocks the CDN, or the PM asked for no external dependency), clamp CJK to the weights
  those faces actually have: map 500 and 600 down to 400, and 800 to 700. Latin keeps
  the full ladder. Log the clamp in `validation_log` — the PM should be told the
  hierarchy is carried by size and spacing on that build, not by weight.
- If the project **self-hosts** the CJK face instead, note in `component-guide.md` that a
  full Traditional Chinese font is several MB *per weight*: either subset the build to
  the glyphs actually shipped, or carry at most two weights (400 and 700).

**3. Two families is a cost decision, not a style decision.** `minimal-elegant` names a
serif for display. Loading a second *CJK* family for headings only is rarely worth it:
apply `cjk_display` only when the project self-hosts and subsets it to the display strings
actually used. Otherwise emit the serif for Latin display and let CJK display stay on the
sans — an editorial serif on Latin headings over a clean CJK sans is a legitimate pairing,
and it is the one that ships.

**4. Tabular figures.** If the finance extension is active, the resolved Latin face must
have real tabular figures (the preset's `tabular_required`). Verify the chosen face
supports `font-variant-numeric: tabular-nums`; if the build is system-only, note that
`-apple-system` does support it but not every Windows fallback does, and that price and
volume columns must therefore also be right-aligned with a fixed column width rather than
relying on the font alone.

Write into the manifest as `typography.font_family.{body,display,mono?}` (each a complete
ordered stack string), `typography.font_source` (`"google" | "self-hosted" | "system"`),
and `typography.webfont_weights` (what the adapter should actually request).

## Step 4f — Resolve the three user-selectable text sizes

Every kit ships a 小 / 中 / 大 text-size setting by default. It is not an add-on the PM
opts into: a product whose text cannot be enlarged is one a whole class of users cannot
read, and retrofitting the control later means auditing every screen that hardcoded a
resolved value in the meantime.

**1. Three steps, one ratio.** Do not invent per-level multipliers. Take the resolved
`base` from Step 4b and scale it, then re-derive every level through the *same*
`effective_ratio`, so the hierarchy the PM chose survives at all three settings:

| 段 | base 倍率 | base (from 16) |
|----|-----------|----------------|
| 小 | ×0.875 | 14 |
| 中 | ×1.000 | 16 — the default |
| 大 | ×1.125 | 18 |

For the finance/web example (ratio 1.200) that resolves to:

| | 小 | 中 | 大 |
|---|---|---|---|
| display | 29 | 33 | 37 |
| h1 | 24 | 28 | 31 |
| h2 | 20 | 23 | 26 |
| h3 | 17 | 19 | 22 |
| body / button | 14 | 16 | 18 |
| label / caption | 12 | 13 | 15 |
| 報價大字 (`^5`) | 35 | 40 | 45 |

±12.5% is deliberate. A smaller step is not worth a setting — a user who reaches for it
wants a difference they can see, and ±1px on body is not one. A larger step starts
breaking layouts that were composed at 中, which turns an accessibility feature into a
bug report.

**2. Floors and caps apply per step, not once.**

- `caption` and `label` never resolve below **12px**. Below that CJK glyphs lose stroke
  separation at normal viewing distance. If 小 would go under, clamp and log it.
- The mobile `display` cap (40) and numeric cap (52) from Step 4b are re-checked at 大.
- Re-run Step 4d's grid snap **per step**. It will snap at some steps and not others — 小
  lands on 23.8px and snaps to 24; 中 lands on 27.2 and snaps to 28; 大 lands on 30.6,
  which is 1.4px from the nearest gridline, so it keeps 30.6. That inconsistency is the
  rule working, not a defect to smooth over.

**3. What does *not* scale.** Only type and its leading. Spacing, radius, elevation and
control heights stay fixed — spacing is a separate axis (information density, Step 4), and
scaling everything together is page zoom, which the browser already does better. The one
exception is a control that can no longer contain its own label at 大: raise that control's
height for that step alone, and log it. Verify this rather than assuming — at the default
heights (32/40/48) an 18px label at leading 1.20 needs 38px including padding, so `md`
holds at all three steps.

**4. Native is layered, not replaced.** On iOS and Android this control sits **on top of**
the OS text-size setting; it does not implement or substitute for Dynamic Type. Say so
plainly in `component-guide.md` — a team that believes the in-app control covers
accessibility will skip the OS integration entirely.

Write into the manifest as `typography.scale_steps.{small,medium,large}`, each a full
resolved table (sizes, line heights, and any per-step control-height override), plus
`typography.default_step: "medium"`.

## Step 5 — Resolve platform + component structure

- Pull the canonical component list/variants/states from `modules/component-structure.md`
  (unchanged structure, just referenced).
- For each platform in `intake_answers.platforms`, note which
  `platform-conventions/{ios,android}.md` file(s) apply. Web has no platform-convention
  file — its conventions live directly in the adapter.
- Resolve control sizing. Write `components.control_height` (`sm`/`md`/`lg` from
  **Control sizing**) and `components.min_hit_target` — the **largest** floor across the
  selected platforms, since one build ships to all of them: 48 if Android is included,
  else 44 if iOS is, else 24 for web-only (WCAG 2.2 SC 2.5.8, Level AA). Then compute each
  control's vertical padding from its height and the level's resolved line box, and write
  those too — adapters must not re-derive padding from the spacing scale.

## Step 6 — Validate contrast

Run `validators/contrast-check.md` over the resolved manifest. Note that a semantic color
is five roles (`fill` / `graphic` / `tint` / `text` / `action`), each with its own
threshold — the presets already carry them; the brand primary from Step 2 must have the
same five resolved here, since a PM-supplied brand color arrives as a single hex.

The correction order matters and is not the obvious one: **change the foreground before
changing the color.** A vivid brand color that fails with white text usually passes with
dark ink, so the brand survives intact. Darkening the hue first is what turns a whole
palette muddy to fix a problem that existed in one place. `fill` is never corrected —
decorative fills are exempt under SC 1.4.11.

Every correction is logged in `manifest.validation_log` with the role, both values and
both ratios.

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
    "scale_basis": { "density": "dense | standard | sparse",
                     "product_type": "...",
                     "base_ratio": 0 },
    "effective_ratio": "base_ratio after the product-type floor and the mobile cap (Step 4b)",
    "scale_px": { "caption": 0, "body": 0, "h3": 0, "h2": 0, "h1": 0, "display": 0,
                  "numeric_current_price_inline": 0, "numeric_current_price_detail": 0, "numeric_market_data": 0 },
    "weight": { "display": 0, "h1": 0, "h2": 0, "h3": 0,
                 "body": 0, "caption": 0, "button": 0, "label": 0 },
    "letter_spacing_bias": "preset value (kept for reference)",
    "letter_spacing_em": { "display": 0, "h1": 0, "h2": 0, "h3": 0,
                           "body": 0, "caption": 0, "button": 0, "label": 0 },
    "line_height_bias": "preset value (kept for reference)",
    "line_height": { "display": 0, "h1": 0, "h2": 0, "h3": 0,
                     "body": 0, "caption": 0, "button": 0, "label": 0 },
    "line_height_px": { "body": 0 },
    "font_family": { "body": "ordered stack string", "display": "ordered stack string" },
    "font_source": "google | self-hosted | system",
    "webfont_weights": { "latin": [400, 500, 600, 700], "cjk": [400, 700] }
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
   - "字太大/太小/標題太誇張" → re-run Step 4b. Tell the PM plainly that this one reflows
     every screen — line counts, column widths and wrap points all move — unlike a weight or
     tracking change. If what they actually mean is "畫面太擠", that is Step 4 (density), and
     it is the cheaper fix; ask which one they want before rewriting the scale.
   - "字太細/太粗/標題不夠重" → adjust `typography.weight` for the named levels only,
     via Step 4c (do not swap the whole preset just to change weight). On a mobile CJK
     build, "標題不夠重" usually cannot be solved by raising the number — Step 4c's 600
     ceiling is a limit of the system font, not a preference. Offer the two real options:
     bundle a CJK font with more weights, or raise the level's size instead.
   - "字距太擠/太開" → re-run Step 4c's letter-spacing table only, keeping the CJK guard
   - "字體換成 X / 不要外部字型 / 公司內網載不到" → re-run Step 4e only, and re-check
     the weight clamp: dropping to a system-only build changes which weights survive
   - "行距太擠/太鬆/一整片字看不下去" → re-run Step 4d only. Note this is NOT the same
     request as "太擠/太鬆" about density (Step 4), which moves spacing between elements;
     if the PM's wording doesn't separate the two, ask one multiple-choice question
   - anything ambiguous → ask ONE clarifying multiple-choice question, don't guess silently
3. Re-run Step 6 (contrast validation) on the changed tokens only.
4. Bump `manifest_version`, rewrite the manifest, re-run the relevant adapter to update
   the actual project files.
5. Tell the PM what changed in one plain sentence — don't re-explain the whole system
   every time.

Never regenerate the entire manifest from scratch on a small tweak — that's how
"team-shared standard" consistency breaks down over time.
