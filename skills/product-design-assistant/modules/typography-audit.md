# Module: Typography Audit

Goal: find every font size and weight in the PM's **existing** screens that is *not* on the
resolved type scale, and offer to snap it back. This is the piece that actually fixes a messy app —
the build flow only *emits* the correct scale (`tokens.*`), it never rewrites the screens
the PM already wrote, so hardcoded sizes like `fontSize: 15` / `fontSize: 22` survive
untouched and keep looking "obviously off". This module closes that gap.

It is a focused slice of the planned Phase 3 Design Review — typography only, no color or
spacing yet. Say that plainly if the PM expects a full review.

## When this runs

- **On demand, with a manifest**: PM says "字級看起來還是亂 / 幫我檢查字太大太小的地方 /
  哪些字沒照規範". Start at Step 1; the manifest supplies the allowlist.
- **On demand, without a manifest, on a project that already has screens**: PM says
  "字級大小不知道怎麼調 / 想把字級變正常 / 字重看起來很亂". Start at **Step 0**, which derives
  the scale from the screens themselves. Do not run the build flow first — see Step 0 for why.
- **Suggested automatically** right after a fresh build, because a brand-new kit on top of an
  old codebase is exactly when the mismatch is largest. Offer it; don't force it.

## Step 0 — No manifest: derive the scale from what is already there

Only for the entry above. The instinct is to run the build flow and audit against its
preset scale, and it is wrong: a preset scale is calibrated for a product being designed,
not for one already written. Auditing an existing site against it turns almost every size
into a finding, and the PM is handed a rewrite instead of a fix.

Derive instead. What the codebase already does is evidence of intent, and most of that
intent is correct — the disease is usually a handful of one-off values, not the whole scale.

**1. Collect sizes with their frequency.** Run the Step 2 scans, but count occurrences per
value, do not just list them. Frequency is the signal that separates a scale from a
mistake: a size used 180 times is the body text; a size used once is a slip. Ignoring
frequency is how an audit ends up "correcting" the most-used size on the site.

**2. Cluster.** Sort the distinct values and merge anything within ~1.5px of a neighbour,
weighting each cluster's centre by occurrence count. `15, 16, 16, 16, 17` is one level at
16, not three levels.

**3. Anchor the body size.** The highest-occurrence cluster between 14–18px (web) or
14–17pt (mobile) is `base`. This is almost always right, and it matters more than every
other choice here: the whole ladder is expressed relative to it.

**4. Fit a ratio, but do not force one.** Test 1.200 / 1.250 / 1.333 against the cluster
centres and report which fits best and how well. If no ratio fits within ~2px across the
levels, **keep the observed cluster centres as the scale** and say so. A tidy ratio the
site does not actually follow is worth less than an honest description of the site.

**5. Do the same for weight.** Collect weights with frequency, cluster them onto the ladder
the codebase actually uses. Flag two things specifically: values that are not multiples of
100 (`550`, `650` — almost always a typo that renders as something else), and more than
four distinct weights, which no design needs and no font ships.

**6. Report the shape before proposing anything:**

```
掃描 42 個檔案,找到 17 種字級、6 種字重

其中 5 種字級佔了 92% 的用量:
  16px ×184  字最多,判定為內文
  14px ×97   說明文字
  20px ×41   小標
  24px ×22   標題
  32px ×8    大標
剩下 12 種各出現 1–3 次,合計 8% ← 這是亂源

級距檢定:這五級最接近 1.250,平均偏差 0.8px
建議收斂成 6 級,並把那 12 個一次性數值歸位
```

**7. Write a typography-only manifest** once the PM confirms, so the derived scale becomes
the single source of truth and later work (colors, spacing, a full kit) can build on it
rather than starting over. Mark it `"derived_from": "existing-source"` so a future run knows
this scale was measured, not chosen — an iteration that changes it is changing the PM's own
site, and should say so.

Then continue at Step 2 with this scale as the allowlist.

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

Weights, on every stack — the same one-off problem happens here and is easier to miss,
because a wrong weight looks like a design choice rather than a mistake:

```bash
grep -rnE "fontWeight:\s*['\"]?[0-9]{3}" src              # RN / JS objects
grep -rnE 'font-weight:\s*[0-9]{3}' src                   # raw CSS
grep -rnE 'font-(thin|light|normal|medium|semibold|bold|extrabold|black)\b' src   # Tailwind
grep -rnE 'font-\[[0-9]{3}\]' src                         # Tailwind arbitrary weight
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

Weights are classified the same way against `typography.weight`, with two extra rules:
`bold` in a CJK build resolves to the mobile ceiling (600) rather than 700 — see
`preset-blender.md` Step 4c — so a literal `700` on a mobile CJK screen is a finding even
though 700 exists in the ladder. And a weight that is not a multiple of 100 is always a
finding: the browser rounds it to a real weight, so the number in the source is a lie about
what ships.

Also surface one **smell metric**: the count of *distinct* raw sizes and weights found. A
screen using 9 one-off sizes is the real disease; the individual snaps are the symptom.

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
