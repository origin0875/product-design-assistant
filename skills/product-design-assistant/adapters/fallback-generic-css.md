# Adapter: Fallback (Generic CSS Variables)

Used when `project-detection.md` can't match a supported stack (Vue, Flutter, native
iOS/Android, empty project, or anything else not yet built). Guarantees the PM always
gets a usable output — never blocks on "unsupported stack."

## Files written

### 1. `.design/tokens.css`

Identical structure to the CSS variables layer in `adapters/web-tailwind.md` — plain CSS
custom properties, framework-agnostic. Any stack can consume this with minimal glue
code. That includes the `--font-weight-*` and `--letter-spacing-*` blocks: a stack
consuming raw tokens has no other source for them.

For `tokens.json`, note that consumers without `em` support (native iOS/Android, Flutter)
need letter-spacing in absolute units — carry the `em` value through as a string and state
the conversion (`em × fontSize = pt`) in `component-guide.md`, rather than silently
emitting a number whose unit is ambiguous.

### 2. `.design/tokens.json`

The same values as flat JSON, for stacks where CSS variables aren't natural (native
mobile, Flutter):

```json
{
  "color": { "primary": { "50": "#...", "500": "#...", "900": "#..." }, "background": "#...", "...": "..." },
  "typography": { "display": { "fontSize": 34, "fontWeight": 700, "letterSpacing": "-0.02em", "lineHeight": 1.15 }, "...": "..." },
  "spacing": [4, 8, 12, 16, 24, 32, 40, 48],
  "radius": { "button": 6, "card": 6, "...": "..." },
  "elevation": [ { "...": "..." } ]
}
```

### 3. `.design/design-starter-kit.md`

The plain-language explanation from `modules/summary-doc.md`, plus an explicit
engineering note at the top:

> "這個專案目前沒有自動整合(例如 Flutter / 原生 iOS / 原生 Android),所以這裡提供的是
> 通用格式的設計變數(`tokens.css` 與 `tokens.json`),需要工程師手動接入專案的樣式系統。
> 之後如果 Skill 支援了你的技術棧,同一份 `.design/design-manifest.json` 可以直接重新產
> 生對應的整合檔案,不需要重新做一次訪談。"

## Do not

- Do not guess at framework-specific syntax for an unsupported stack (e.g. don't write a
  half-correct Flutter `ThemeData` — that's worse than plain tokens, it looks integrated
  but isn't validated)
- Do not skip the contrast validation step just because output is "just tokens" — the
  manifest passed validation before this adapter ever runs

## Idempotency

Same as other adapters — `tokens.css`/`tokens.json`/`design-starter-kit.md` are fully
regenerated on every run from the current manifest.
