# Platform Convention: Android

Applies on top of the shared Foundation (color/typography/spacing/radius from the
manifest) and the structural contract in `modules/component-structure.md`. This file
only overrides **interaction pattern and placement** — never colors or type scale. Do
not copy Material Design's actual spacing/color/component values; only the interaction
conventions users already expect from Android.

## Touch targets

- Minimum hit target: **48 × 48 dp** — larger than iOS's 44, so a shared React Native
  build must satisfy 48 or split the value with `Platform.select`.
- Same exception and same method as the iOS file: an ergonomic floor, reached with
  `hitSlop` rather than by growing the visual box.

## Navigation

- Back navigation: system back gesture/button (edge swipe or nav-bar back) — do not
  rely on an in-app back button as the only way back, the system-level back must always
  work
- Root screens: top app bar, title left-aligned (not centered)
- Trailing action(s) in top app bar: max 2 icons, overflow (3rd+ action) goes into a
  "more" (⋮) menu — do not just crowd more icons in

## Bottom Navigation

- Can hide on scroll-down / reappear on scroll-up for content-heavy feeds (more common
  here than on iOS)
- Selected tab: label + icon both always visible (Android convention keeps labels
  visible even when unselected, unlike some iOS apps that fade inactive labels)

## Bottom Sheet

- `action-list` variant → Material-style modal bottom sheet: full-width list of actions,
  no separate detached "Cancel" row — dismiss via tap-outside or swipe-down instead
- `content` variant → standard modal bottom sheet, supports collapsed/expanded states
  via drag handle

## Dialog

- `alert` / `confirmation` variants: actions stacked or side-by-side text buttons
  bottom-right of the dialog (not centered like iOS), primary action on the far right
- Destructive confirmation dialogs must state the consequence in the body text
  explicitly (Android users expect the dialog body to spell out what will happen, not
  just a title)

## Gestures & feedback

- Swipe-to-dismiss on list rows for destructive actions is acceptable but less universal
  than on iOS — always pair with a visible affordance (icon) as well, not gesture-only
- Ripple effect on all tappable surfaces (buttons, list rows, cards with `interactive`
  variant) — this is the baseline Android tap feedback, always include it
