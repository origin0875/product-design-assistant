# Platform Convention: iOS

Applies on top of the shared Foundation (color/typography/spacing/radius from the
manifest) and the structural contract in `modules/component-structure.md`. This file
only overrides **interaction pattern and placement** — never colors or type scale. Do
not copy Apple HIG's actual spacing/color/component values; only the interaction
conventions users already expect from iOS.

## Touch targets

- Minimum hit target: **44 × 44 pt**.
- This is a deliberate exception to the "no values" rule above: it is an ergonomic floor
  set by the size of a finger, not a style value, and it differs between the two platforms
  — which is precisely the kind of thing this file exists to carry.
- Reach it with `hitSlop`, not by enlarging the control's visual box (see **Control
  sizing** in `modules/component-structure.md`).

## Navigation

- Back navigation: leading-edge back button/chevron + swipe-from-left-edge gesture —
  always keep the swipe gesture enabled, iOS users rely on it
- Root screens: large title that condenses on scroll (if content is scrollable)
- Trailing action(s) in nav bar: max 2, icon-only acceptable here (unlike bottom nav)

## Bottom Navigation (Tab Bar)

- Persistent, always visible (does not hide on scroll unless the app is
  media/immersive-first)
- Selected tab: icon fills in / color shifts to primary — no underline indicator (that's
  an Android/web pattern)

## Bottom Sheet / Action List

- `action-list` variant → native iOS action sheet convention: actions stacked,
  destructive action styled in `error` color, a separate "Cancel" action visually
  detached from the group (not just another list item)
- `content` variant → sheet with a drag handle at top, supports partial-height +
  full-height drag states

## Dialog

- `alert` / `confirmation` variants: centered, compact, max 2 actions side-by-side
  (primary action on the right/trailing side)
- Avoid stacking more than 2 buttons in an alert — if more actions are needed, use the
  action-list bottom sheet instead

## Gestures & feedback

- Swipe-to-delete on list rows is the expected pattern for destructive actions on list
  items (in addition to, not instead of, an explicit confirmation for anything
  irreversible)
- Haptic feedback on: primary button confirm, toggle switches, pull-to-refresh trigger
