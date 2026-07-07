# Module: Summary Doc

Goal: produce `.design/design-starter-kit.md` — the document the PM actually reads.
Everything else in this Skill produces engineering artifacts; this is the one PM-facing
deliverable. Zero jargon. No "Design Token", "Semantic Token", "Foundation",
"Variables", "8pt Grid" anywhere in this document.

## Structure

```md
# {product_name} 設計規範

## 這套規範是什麼調性
{principle.design_keywords 白話呈現，例如「專業、穩重、可信賴」}

我們依照這個調性做了以下設計決定：
{principle.ux_principles 逐條列出，白話版}

## 顏色
{展示每個顏色色塊 + 用途說明，例如：
"主色（深藍）：用在主要按鈕、重要連結、選中狀態"
"成功色（綠）：用在完成/成功訊息"
不寫 hex code 給 PM 看的必要性低，但可以附上方便工程師對照}

## 文字大小
{用「標題／內文／說明文字」等白話分類展示各級文字，不寫「H1/H2/scale ratio」這類詞，
用視覺範例呈現大小差異}
{如果啟用了金融擴充，額外說明「數字排版」：「價格、漲跌幅這類數字會用固定寬度對齊的字體，
方便使用者快速掃視比對」}

## 元件長相
{列出 Button/Input/Card/Bottom Navigation/Search/Dialog/Toast/Empty State/Loading，
每個附上一句話說明何時用、長什麼樣，不列 props/variants 這種工程用語清單 —
那些留在 adapter 產出的 component-guide.md 給工程師看}

## 畫面留白與版面
{用「頁面邊界留多少空間」「區塊之間隔多開」這種白話描述，不寫 spacing scale 數字表}

## 以後怎麼調整
"如果想調整任何東西，直接跟我說就好，例如：
- 「品牌色改成藍色」
- 「按鈕圓角改大一點」
- 「整體感覺再活潑一點」
我會直接更新整套規範，同時同步到你專案裡的程式碼，你不需要自己去改任何檔案。"

## 給工程師的技術對照文件
{一句話指向 .design/component-guide.md 與 .design/tokens.* — 不需要 PM 理解內容，
但要知道這些檔案存在、給工程師接手用}
```

## Tone rules

- Every sentence should be understandable by someone who has never opened Figma.
- Explain the *why* behind decisions in one clause, not a paragraph — e.g. "深藍色，讓
  使用者一眼覺得這個 App 值得信任" not "藍色在色彩心理學中象徵信任與穩定,根據多項研究..."
- Never expose the preset name (e.g. "professional-trustworthy") — that's internal
  implementation, translate to `principle.design_keywords` instead.

## When this runs

Once after the initial build (Step 7 of `preset-blender.md` completes + adapter has
written files), and again after any iteration — but on iteration, only regenerate the
sections actually affected, and add one line at the top: "已更新：{一句話說明這次改了
什麼}".
