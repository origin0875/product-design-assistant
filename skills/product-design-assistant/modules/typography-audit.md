# Module: Typography Audit

Goal: find every font size in the PM's **existing** screens that is *not* on the resolved
type scale, and offer to snap it back. This is the piece that actually fixes a messy app —
the build flow only *emits* the correct scale (`tokens.*`), it never rewrites the screens
the PM already wrote, so hardcoded sizes like `fontSize: 15` / `fontSize: 22` survive
untouched and keep looking "obviously off". This module closes that gap.

It is a focused slice of the planned Phase 3 Design Review — typography only, no color or
spacing yet. Say that plainly if the PM expects a full review.

## When this runs

- **On demand**: PM says something like "字級看起來還是亂 / 幫我檢查字太大太小的地方 /
  哪些字沒照規範". Requires an existing `.design/design-manifest.json` — if there isn't one,
  run the build flow first (there's no scale to audit against yet).
- **Suggested automatically** right after a fresh build, because a brand-new kit on top of an
  old codebase is exactly when the mismatch is largest. Offer it; don't force it.

## Step 1 — Load the allowed scale

Read `typography.scale_px` from the manifest. That set of pixel values (plus the numeric
tiers if the finance extension is active) is the **allowlist**. Everything else is a finding.

## Step 2 — Scan the source (skip generated + vendor files)

Search only the PM's own source. **Exclude**: `.design/`, `src/theme/tokens.*` (generated),
`node_modules/`, `.expo/`, `dist/`, `build/`, `ios/`, `android/` build output, `*.d.ts`.

React Native / JS-TS projects:

```bash
# quote the --include globs — an unquoted *.ts is expanded by zsh and errors when nothing matches
grep -rnE 'fontSize:\s*[0-9]+' src \
  --include='*.ts' --include='*.tsx' --include='*.js' --include='*.jsx' \
  | grep -vE 'theme/tokens\.'
```

Web + Tailwind / CSS projects also check:

```bash
grep -rnE 'font-size:\s*[0-9.]+px' src         # raw CSS
grep -rnE 'text-\[[0-9.]+px\]' src             # Tailwind arbitrary values
```

Ignore sizes that are already token references (`typography.h1.fontSize`, `text-h1`,
`var(--font-size-h1)`) — those are correct by construction.

## Step 3 — Classify each hit

For each raw numeric size found:

- **On scale** → not a finding.
- **Off scale** → finding. Compute the nearest allowed size and the delta:
  - nearest = the `scale_px` entry with the smallest absolute px difference
  - if two are equidistant, prefer the smaller (safer on mobile)
- **Tone the confidence**: a value 1px off a token (`17` vs `16`) is almost certainly a
  rounding slip → high confidence snap. A value far from every token (`44` when the nearest
  are `38` and `52`) is a judgement call → flag it but ask which way to go rather than
  snapping silently.

Also surface one **smell metric**: the count of *distinct* raw sizes found. A screen using 9
different one-off sizes is the real disease; the individual snaps are the symptom.

## Step 4 — Report before touching anything

Show the PM a plain list, grouped by file, e.g.:

```
src/screens/PortfolioScreen.jsx
  L42  fontSize: 22   → 建議 21(小標 h3)     偏差 1px,幾乎確定是手滑
  L57  fontSize: 15   → 建議 16(內文 body)   偏差 1px
  L88  fontSize: 44   → 介於 大標38 和 hero52 之間,要往哪邊靠?(需你決定)
共 3 處、6 種不同字級 → 建議收斂到規範的 6 級
```

Keep it PM-readable: name the token in plain language ("小標", "內文"), not `scale_px.h3`.

## Step 5 — Fix only with confirmation

- Never auto-edit. The PM (or engineer) confirms the batch, same guardrail as any other
  file mutation in this Skill.
- High-confidence snaps (≤2px off a single nearest token) can be offered as "全部套用".
- Ambiguous ones (Step 3 judgement calls) get one multiple-choice question each — never guess.
- **Preferred fix is a token reference, not a new magic number**: replace `fontSize: 22` with
  `fontSize: typography.h3.fontSize` (RN) / `text-h3` (Tailwind), so the value can't drift out
  of scale again on the next edit. Only fall back to writing the raw corrected px when the
  file has no access to the theme import.
- Re-run Step 2 after applying to confirm the findings are cleared, and report the new
  distinct-size count so the PM sees the convergence.

## Guardrails

- This module reads the scale but **never changes it** — if the PM thinks a size *should*
  exist that doesn't, that's an iteration on the manifest (`preset-blender.md`), not an audit
  finding to hardcode.
- Don't report sizes inside comments or generated blocks.
- If zero findings: say so plainly ("字級都在規範上,沒有需要修的") — a clean audit is a
  valid, useful result, not a reason to invent nits.
