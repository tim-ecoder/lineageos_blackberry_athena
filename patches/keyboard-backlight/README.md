# keyboard-backlight

PowerManagerService / PhoneWindowManager fixes so the athena keyboard
backlight follows Settings.Secure.keyboard_brightness (driven by
DeviceSettings slider) all the way to /sys/class/leds/keyboard-backlight.

Both files live in frameworks/base. Apply with:

    cd /data/LOS23/frameworks/base
    git apply /data4/LOS23-build/lineageos_blackberry_athena/patches/keyboard-backlight/*.patch

Local-only — not pushed to tim-ecoder/android_frameworks_base.

## pwm-reannounce-keyboard-visibility.patch

`PhoneWindowManager.applyLidSwitchState()` (called from
`enableScreenAfterBoot`) calls `PowerManager.setKeyboardVisibility(
mHaveBuiltInKeyboard && !isHidden(...))` once at boot. On athena the
call fires BEFORE the input subsystem enumerates the stmpe keypad, so
`mHaveBuiltInKeyboard` is still false and `setKeyboardVisibility(false)`
gets latched. Once `adjustConfigurationLw` later sets
`mHaveBuiltInKeyboard = true`, nothing re-announces the visibility to
PMS, so `mKeyboardVisible` stays stuck at false.

Fix: when `adjustConfigurationLw` flips `mHaveBuiltInKeyboard` from
false to true, re-call `setKeyboardVisibility(isBuiltInKeyboardVisible())`
so PMS sees the keyboard appear.

## pms-drop-keyboard-visible-gate.patch

Defense-in-depth for the above. PowerManagerService's keyboard-backlight
update path gates the write on `mKeyboardVisible` — on a slider phone
with a real LID switch that makes sense (hide backlight when lid
closed), but athena has no lid, the keyboard is always physically
accessible, and we never want the backlight gated off. Drop the ternary
and always use the computed `keyboardBrightness` value (which is
already 0 via `mKeyboardBrightness` and the Settings.Secure override
when the user wants it off).

This is needed in addition to the PWM fix because even with the
visibility announce fix, a boot-time race can still leave
`mKeyboardVisible=false` if the input enumeration ordering shifts.

## kbd-backlight-inverse-lcd.patch (in `../`)

Lives at top-level `patches/kbd-backlight-inverse-lcd.patch` to match
the LOS22 layout. Replaces the linear `screenBrightScale =
max(0.1, brightness/255)` mapping in PowerManagerService.java with a
stepped inverse-LCD curve tuned for athena's physical QWERTY:

| LCD brightness    | kbd backlight |
|-------------------|---------------|
| 0                 | off           |
| 1–3 (darkroom)    | ~47 % (don't blind in pitch dark) |
| 4–49 (dim/medium) | full          |
| 50+ (daylight)    | off (kbd light is invisible outdoors anyway, wastes power) |

Applies cleanly on top of the two patches in this directory.
