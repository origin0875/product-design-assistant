# Product Design Assistant

A Claude Code plugin that lets a Product Manager (no design background required) turn a
plain-language product description into a working design starter kit — colors,
typography, spacing, components, layout — written directly into their Web or mobile app
project.

## Why

PMs planning a new product usually don't know how to start a design system: what type
scale to use, how to pick a color palette, what components should look like, how to keep
screens consistent. This skill handles all of that behind the scenes. The PM never sees
or needs to understand terms like Design Token, Semantic Token, Foundation, Variables, or
8pt Grid.

## 使用說明

**[USAGE.md](USAGE.md)** — 給 PM 的完整使用說明（繁體中文）：兩條路線的流程圖與逐步說明、
七題各自決定什麼、產出的檔案、怎麼用白話調整、給前端的三條規則、以及做不到的事。

## What it does (Phase 1 — current)

1. Detects the PM's project (Web+Tailwind, React Native, or falls back to generic CSS
   variables for anything else).
2. Interviews the PM with plain multiple-choice questions — product type, platform,
   tone, optional brand color, information density.
3. Picks one of 3 calibrated style presets (專業穩重 / 親和活潑 / 簡約優雅), blends in
   the brand color, activates domain extensions (e.g. a finance product automatically
   gets Numeric Typography + Up/Down/Chart/Risk colors), validates every color pairing
   against WCAG AA.
4. Writes the result directly into the project (Tailwind config + CSS variables, or a
   React Native theme module) plus a plain-language `design-starter-kit.md` for the PM
   and a `component-guide.md` for engineers.
5. Shows a live or static preview — never just says "done."
6. Takes follow-up tweaks in plain language ("品牌色改藍色", "按鈕圓角改大一點") and
   updates the whole system consistently from a single source of truth
   (`.design/design-manifest.json`).

There is a **second entry point** that skips all of the above. A PM who already has a
working site and just wants the type fixed ("字級大小不知道怎麼調") goes straight to
`modules/typography-audit.md`, which derives a scale from the sizes and weights the site
already uses — weighted by how often each one appears — and offers to snap the outliers
back. The build flow never touches existing screens, so running it first on such a project
produces a kit the PM did not ask for and leaves the original complaint untouched. See
[USAGE.md](USAGE.md) §1.

## Roadmap (not yet implemented)

- **Phase 2 — Screen Generator**: generate actual app screens (home, detail, search,
  etc.) that follow the Starter Kit automatically.
- **Phase 3 — Design Review**: PM uploads a screen, skill checks it against the Starter
  Kit for inconsistent type/spacing/components and UX issues.

## Structure

```
skills/product-design-assistant/
├── SKILL.md                    # entry point / orchestration
├── presets/                    # 3 calibrated style presets (the quality floor)
├── modules/                    # intake, project detection, preset blending, summary doc, component structure contract
├── adapters/                   # manifest → real project files (per tech stack)
├── platform-conventions/       # iOS vs Android interaction pattern overrides
└── validators/                 # WCAG AA contrast enforcement
```

See `skills/product-design-assistant/SKILL.md` for the full flow and design rationale
(preset structure vs. live-computed brand color blending, manifest-as-source-of-truth,
adapter/platform-convention separation).

## Install

Add this directory as a plugin (`.claude-plugin/plugin.json` is already set up), or copy
`skills/product-design-assistant/` into any project's `.claude/skills/` for local-only
use.
