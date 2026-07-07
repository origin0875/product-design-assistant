# Module: Project Detection

Goal: figure out (a) what's already in the PM's project, and (b) which adapter to use —
without asking the PM anything. PMs don't know what stack they're on; infer it.

## Step 1 — Check for an existing manifest

```bash
test -f .design/design-manifest.json && echo "EXISTING" || echo "NEW"
```

- **EXISTING**: this is an iteration request, not a fresh build. Read the manifest, go to
  the "Iteration" section of `preset-blender.md`. Do not re-run intake.
- **NEW**: continue to Step 2.

## Step 2 — Detect tech stack

Check for these signals, in order, and stop at the first match:

| Signal | Detected stack | Adapter to use |
|---|---|---|
| `package.json` contains `react-native` or `expo` dependency | React Native | `adapters/react-native.md` |
| `tailwind.config.*` exists, or `package.json` has `tailwindcss` dependency | Web + Tailwind | `adapters/web-tailwind.md` |
| `package.json` exists but no Tailwind, has `react`/`next`/`vue`/other web framework | Web, no Tailwind | `adapters/fallback-generic-css.md` |
| `pubspec.yaml` exists | Flutter | not yet supported — see fallback below |
| `*.xcodeproj` / `Package.swift` exists | iOS native (Swift) | not yet supported — see fallback below |
| `build.gradle` + `AndroidManifest.xml` exists | Android native (Kotlin/Java) | not yet supported — see fallback below |
| none of the above / empty project | Unknown | `adapters/fallback-generic-css.md` |

```bash
cat package.json 2>/dev/null | grep -E '"(react-native|expo|tailwindcss|next|react|vue)"'
ls tailwind.config.* pubspec.yaml *.xcodeproj build.gradle 2>/dev/null
```

## Step 3 — Handle unsupported stacks

If detection lands on Flutter / native iOS / native Android (no adapter built yet in
Phase 1): **do not force an unsupported adapter's output into the project.** Fall back to
`adapters/fallback-generic-css.md` (generic CSS variables + a plain-language JSON token
file), and tell the PM plainly:

> "你的專案是 {detected_stack},目前 Skill 還沒有針對這個技術棧的自動整合,所以我先產出一份通用的設計變數檔給你的工程師接入。等之後支援 {detected_stack} 時,同一份 manifest 可以直接重新產生對應的整合檔案,不用重做一次訪談。"

This keeps the promise from `intake.md`/`preset-blender.md`: the manifest is the durable
source of truth, adapters are swappable without re-asking the PM anything.

## Step 4 — Detect existing design artifacts (avoid clobbering PM's/engineer's prior work)

Before writing anything, check whether the target output files already exist with
content that looks intentional (not scaffold defaults):

```bash
test -s tailwind.config.js && echo "tailwind.config.js has content — review before overwrite"
test -d src/theme && echo "src/theme exists — review before overwrite"
```

If existing design-related files look non-trivial, don't silently overwrite. Show the PM
what would change and confirm before writing, same as any other risky file operation.
