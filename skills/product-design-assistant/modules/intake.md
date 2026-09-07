# Module: Intake

Goal: extract everything needed to pick and blend a preset — **using only multiple-choice
questions**. Never ask an open-ended question that requires design vocabulary
(e.g. never ask "what type scale ratio do you want"). If the PM types free text anyway,
map it to the closest curated option yourself — don't relay jargon back to them.

Run this only when no `.design/design-manifest.json` exists yet in the project, or when
the PM explicitly asks to start over. If a manifest exists, skip to the iteration flow in
`preset-blender.md` instead.

## Questions to ask (via AskUserQuestion, batched in as few rounds as possible)

Ask in two batches so the PM isn't hit with everything at once.

**Batch 1 — the product**
1. 產品名稱是什麼?(free text — this is just a label, not a design decision)
2. 產品類型是什麼? Options: 金融/投資理財, 電商/購物, 社群/社交, 內容/媒體閱讀, 工具/生產力, B2B後台/企業系統, 其他(請描述)
   - This answer feeds `product-type extension` matching in `preset-blender.md` (e.g. "金融/投資理財" → finance extension).
3. 平台是? Options: Web, iOS App, Android App, iOS + Android (同一套), 全部都要
4. 主要使用者是誰?(free text, 1 句話即可 — used only for the PM-facing summary doc, not for token math)

**Batch 2 — the feel**
5. 如果要用一句話形容這個產品給人的感覺,比較接近哪一種? Options (map 1:1 to presets, label in plain PM language, no jargon):
   - "專業、穩重、值得信賴" → `professional-trustworthy`
   - "親切、活潑、有溫度" → `friendly-playful`
   - "簡約、優雅、有質感" → `minimal-elegant`
   - "我不確定,你幫我判斷" → infer from product type answer (finance/B2B → professional-trustworthy; social/lifestyle/ecommerce → friendly-playful; content/reading/premium → minimal-elegant)
6. 有沒有指定的品牌主色?(如果有品牌 Logo 或指定色,直接說色票或描述,例如「深藍色」「跟 Logo 一樣的橘色」；沒有的話選"沒有,用系統建議的顏色即可")
7. 這個產品的資訊密度高不高? Options: "很多數據/列表,使用者會頻繁掃描比對"(dense), "適中,一般 App 的量"(standard), "很簡潔,一次只看少量內容"(sparse)
   - Maps to `component_bias.density` override on top of the preset default (Step 4),
     **and** selects the type scale ratio (Step 4b). It is the single information-density
     input, so keep it phrased in terms of how the PM's users read the screen — do not
     narrow it into a whitespace-only question.

## Output of this module

A structured `intake_answers` object, e.g.:

```json
{
  "product_name": "...",
  "product_type": "finance",
  "platforms": ["ios", "android"],
  "primary_users": "...",
  "tone_choice": "professional-trustworthy",
  "brand_color": { "provided": true, "value_or_description": "深藍色" },
  "density_override": "dense"
}
```

Pass this directly to `preset-blender.md`. Do not ask the PM to confirm intermediate
design terms — the next thing they should see is the finished Starter Kit, not a
checklist of jargon to approve.
