# pin-lag

Fix for slow PIN entry on the lockscreen on LOS 23.2 (Android 16).

## Symptom

PIN keypad on the lockscreen renders at ~12.5 fps (~78 ms/frame, 5× over
the 16 ms budget) on athena (BlackBerry KEY2, SDM660, Adreno 512). Each
tap takes ~570 ms to render the PIN dot. LOS 22.2 (Android 15) on the
same hardware does ~60 fps with ~30 ms tap-to-dot — fast and smooth.

## Cause

Android 16 enables three SystemUI aconfig flags that activate new bouncer
rendering code paths:

- `bouncer_ui_revamp` — cosmetic blur
- `bouncer_ui_revamp_2` — button animations, font changes, and per-frame
  PIN dot color resolution (this is the dominant cost)
- `pin_input_field_styled_focus_state` — styled focus state on PIN input

All three are set `state: ENABLED, permission: READ_ONLY` in the `bp3a`
release config that LineageOS's `bp4a` inherits from. `READ_ONLY` means
runtime `device_config put` cannot flip them; the flag values are baked
into SystemUI bytecode at build time.

The slow paths gated by `bouncer_ui_revamp_2` live in:

- `frameworks/base/packages/SystemUI/src/com/android/keyguard/NumPadAnimator.java`
  — entire color resolution path replaced with per-call `BouncerColors`
  lookups; new `Animation.expansion*` constants
- `frameworks/base/packages/SystemUI/src/com/android/keyguard/PinShapeHintingView.java:121`
  — PIN dot color resolved per redraw via `getPinHintDotColor()` /
  `getPinShapeColor()`, each checking the flag again
- `frameworks/base/packages/SystemUI/src/com/android/systemui/bouncer/ui/BouncerColors.kt`
- `frameworks/base/packages/SystemUI/src/com/android/systemui/bouncer/ui/BouncerMessageView.kt`

LOS 22.2 (A15) does not enable any of these flags in any release config —
they default off, the new code paths never execute, and the legacy XML
keyguard renders at ~60 fps.

## Fix

Add three textproto overrides under
`vendor/lineage/release/aconfig/bp4a/com.android.systemui/` setting all
three flags `state: DISABLED, permission: READ_ONLY`. The pre-existing
`Android.bp` in that directory globs `*_flag_values.textproto` so no
`Android.bp` edit is needed.

Disabling at build time triggers aconfig dead-code elimination — the
patched SystemUI APK is 3,479 bytes smaller than the baseline.

## Apply

    cd /data/LOS23/vendor/lineage
    git apply /data/LOS23/.../patches/pin-lag/disable-bouncer-revamp-flags.patch
    # or, from this repo:
    cd /data/LOS23/vendor/lineage
    git apply <path-to>/lineageos_blackberry_athena/patches/pin-lag/disable-bouncer-revamp-flags.patch

Then rebuild SystemUI (or full image):

    mka SystemUI                          # ~2 min on incremental
    # or full system.img via build-athena-all.sh

## Verification

On athena 5000194162, post-flash:

- 6-tap PIN entry total: 1.04 s including unlock (was ~3.4 s baseline)
- `KEYGUARD_DONE_DRAWING` storms gone — bouncer renders on legacy XML path
- User-perceived lag absent: "pin is typed fast"

## Cross-reference

- GrapheneOS issue #6618 documents the same A16 SystemUI Compose-migration
  regression on Pixel devices — this is an upstream AOSP issue
- Likely benefits other LOS 23 devices on weak GPUs (SDM660 siblings,
  SDM845-era hardware)

## Files

- `disable-bouncer-revamp-flags.patch` — adds the three textproto files
