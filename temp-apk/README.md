# PIN bouncer rendering A/B SystemUI APKs (athena LOS 23.2)

Built 2026-05-08 from `frameworks/base @ lineage-23.2` against
`vendor/lineage @ lineage-23.2` with kernel `sdm660-4p19`. Both APKs
target `com.android.systemui` for `lineage_athena-bp4a-userdebug`.

## Files

| APK | aconfig flag state |
|---|---|
| `SystemUI_baseline_no_overrides.apk` | upstream LOS 23.2 — `bouncer_ui_revamp=ENABLED`, `bouncer_ui_revamp_2=ENABLED`, `pin_input_field_styled_focus_state=ENABLED` (all `READ_ONLY`, baked in via `bp3a` inheritance) |
| `SystemUI_revamp_disabled.apk` | three-flag override — all three `DISABLED, READ_ONLY` via `vendor/lineage/release/aconfig/bp4a/com.android.systemui/*_flag_values.textproto` |

## Hypothesis

LOS 23 (Android 16) introduced `bouncer_ui_revamp_2` which gates new
slow code paths in:

- `frameworks/base/packages/SystemUI/src/com/android/keyguard/NumPadAnimator.java`
  — entire color resolution path & animation duration constants replaced
- `frameworks/base/packages/SystemUI/src/com/android/keyguard/PinShapeHintingView.java:121`
  — PIN dot color resolved per-frame via `getPinHintDotColor()` /
  `getPinShapeColor()` checking the flag every redraw
- `frameworks/base/packages/SystemUI/src/com/android/systemui/bouncer/ui/BouncerColors.kt`
- `frameworks/base/packages/SystemUI/src/com/android/systemui/bouncer/ui/BouncerMessageView.kt`

This causes keyguard to render at ~12.5 fps (78 ms/frame, 5× over
budget) on athena (Adreno 512), making PIN entry feel laggy.
LOS 22.2 (A15) does not enable any of these flags — bouncer renders
at ~60 fps.

## Acceptance test

Lock device, capture logcat, fast 6-tap PIN entry. Compute:

- Median KGD-to-KGD interval (target ≤ 30 ms; ideal ~16 ms)
- WARM tap haptic→1st-frame delta (target ≤ 30 ms; baseline 67–77 ms)
- COLD tap haptic→1st-frame delta (target ≤ 60 ms; baseline 195–220 ms)

If `SystemUI_revamp_disabled.apk` meets all three, the regression is
in this flag set. Then re-enable `bouncer_ui_revamp` (cosmetic blur)
to find the minimal disable set.

## Install (privileged system app)

`adb install -r` will fail. Use one of:

1. Sideload via `adb push` to `/system/system_ext/priv-app/SystemUI/SystemUI.apk`
   after `mount -o rw,remount /system_ext` (requires patched fstab or
   debugfs). On A/B, write to inactive slot then switch.
2. Repackage as Magisk module overlaying `/system_ext/priv-app/SystemUI/SystemUI.apk`.
3. Replace in vendor/lineage build, build full system.img, fastboot
   flash system.

## Notes

- Both APKs are unsigned-from-source builds (LineageOS test-keys).
  Do not mix with release-key system.img.
- Files are temporary — once the fix is verified and either upstreamed
  or committed as a permanent device-tree override, this `temp-apk/`
  folder will be removed.
