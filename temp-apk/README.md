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

---

# BT-lag-on-unlock A/B BT APEXes (athena LOS 23.2)

Built 2026-05-09 from `packages/modules/Bluetooth @ lineage-23.2`
against the same `vendor/lineage @ lineage-23.2` tree, kernel
`sdm660-4p19`. Both APEXes contain the `com.android.bluetooth`
mainline module with different aconfig flag states baked in.

## Files

| APEX | aconfig flag state |
|---|---|
| `com.android.bt_baseline.capex` | upstream LOS 23.2 BT module — `a2dp_fmq_read_exact=ENABLED`, `a2dp_source_null_fixed_queue=ENABLED`, `ref_counted_native_wakelock=ENABLED`, `a2dp_sbc_underflow_recovery=ENABLED` (all `READ_ONLY`, baked via `bp3a` inheritance) |
| `com.android.bt_4flag_disabled.capex` | four-flag override — all four `DISABLED, READ_ONLY` via `vendor/lineage/release/aconfig/bp4a/com.android.bluetooth.flags/*_flag_values.textproto` |

md5:
- `6bbaa0bb931a6e396a340b82155191f5  com.android.bt_baseline.capex`
- `3f991a6dd8bbd22fb5204d1a7d60fea1  com.android.bt_4flag_disabled.capex`

(Same compressed size, different content — aconfig payload is small but
DCE strips reachable code in the disabled-flag build.)

## Hypothesis

LOS 23 (Android 16) enables four `com.android.bluetooth.flags` which
activate new code paths in:

- `packages/modules/Bluetooth/system/btif/src/btif_a2dp_source.cc:335,441`
  (`a2dp_source_null_fixed_queue`) — A2DP source init does RT scheduling
  enable + `audio::a2dp::init()` synchronously on main thread instead of
  posted via `do_in_main_thread(btif_a2dp_source_startup_delayed)`
- `packages/modules/Bluetooth/system/btif/src/btif_a2dp_source.cc:203,573`
  (`ref_counted_native_wakelock`) — `wakelock_release()` only fires when
  currently streaming; otherwise wakelock kept across stream stop/restart
- `packages/modules/Bluetooth/system/audio_hal_interface/aidl/a2dp/client_interface_aidl.cc:540`
  (`a2dp_fmq_read_exact`) — new `ReadAudioDataExact` path with 10-retry
  loop returning 0 on underflow
- `a2dp_sbc_underflow_recovery` — gates feeding-counter reset on SBC
  underflow (codec on test pairing is AAC, included for safety)

LOS 22.2 (A15) has none of these flags overridden — they default off in
the Mainline BT module, the new code paths never execute, and BT
audio resumes within ~250ms of unlock without an audible glitch.

## Background — what was already disproven

Earlier today the build session tested three source-level patches in
`aidl/a2dp/client_interface_aidl.cc::ReadAudioData`:
1. Pad short reads with PCM silence in the new ExactRead path
2. Pad short reads in the legacy path
3. Force `a2dp_fmq_read_exact` flag OFF in source via `false &&`

Variant 3 was decisive: the legacy A15 code path runs identically on
LOS23 hardware and the lag persists. Conclusion at the time: regression
is upstream of the FMQ producer. **All three source patches were
reverted; tree is clean.**

The four-flag aconfig override approach is the next test, modeled on
the successful PIN-lag fix earlier today (`SystemUI_revamp_disabled.apk`
above). Architectural pattern matches: A16 introduces flags `ENABLED,
READ_ONLY` that activate new code paths, LOS22 baseline has them off,
runtime override is cosmetic because READ_ONLY bakes the value into
bytecode at compile time.

## Acceptance test

Lock device, BT music playing, wait 30s+ for the bug to arm via deep
sleep, unlock, listen for the single audible glitch at unlock moment.

Pre-flash baseline: every unlock after first deep sleep glitches.
Target: zero audible glitch on any unlock.

Codec on test pairing: AAC 44100 Hz / 16-bit / stereo (verified via
`dumpsys bluetooth_manager | grep mCodecConfig`).

## Install

The BT module is a privileged mainline APEX. Three install paths:

1. **Full system flash via fastboot** (cleanest):
   `adb reboot bootloader && cd /data4/LOS23-build && echo "user" | sudo -S bash flash-imgs-all.sh`
   This is what the build session recommends — no APEX-validation issues.

2. **APEX activation via apexservice** (testable but can fail prod
   verification):
   ```
   adb push com.android.bt_4flag_disabled.capex /data/local/tmp/
   adb shell 'cmd apexservice activatePackage /data/local/tmp/com.android.bt_4flag_disabled.capex'
   adb shell stop && adb shell start
   ```

3. **Replace in /system/apex/** (requires `mount -o rw,remount /` or
   patched fstab; also won't survive verified boot on user builds).

## Verification on device after install

```
cmd device_config get com.android.bluetooth a2dp_fmq_read_exact
cmd device_config get com.android.bluetooth a2dp_source_null_fixed_queue
cmd device_config get com.android.bluetooth ref_counted_native_wakelock
cmd device_config get com.android.bluetooth a2dp_sbc_underflow_recovery
```

All four should return `false` AND be honored by code (READ_ONLY baked
at compile time means runtime cannot flip them back).

---

# BT-lag round 2 — audio framework aconfig (athena LOS 23.2)

Built 2026-05-09 from `frameworks/base @ lineage-23.2` against the same
`vendor/lineage @ lineage-23.2` tree. Round 1 (BT APEX 4-flag disable)
ruled out the BT module flag space — lag persisted unchanged. Audio
framework is the next angle.

## File

| File | What |
|---|---|
| `services_audio_flags_disabled.jar` | rebuilt `/system/framework/services.jar` with two `com.android.media.audio` flags forced DISABLED, READ_ONLY at compile time. md5 `8ab9b00c42361b8fdffd1c82eaec23a9`. |

The two disabled flags:

- **`optimize_bt_device_switch`** (was ENABLED via bp3a inheritance)
- **`as_device_connection_failure`** (was ENABLED via bp3a inheritance)

## Hypothesis (strong)

`AudioDeviceInventory.java::setBluetoothActiveDevice` line ~2280:

```java
if (!info.mSupprNoisy
        && (... A2DP/HEARING_AID/LE_AUDIO ...)
        && !info.mIsDeviceSwitch) {            // ← LOS22 path: branch taken
    delay = checkSendBecomingNoisyIntentInt(...);
} else {
    delay = 0;                                  // ← LOS23 with optimize_bt_device_switch=true: 0 delay
}
```

`info.mIsDeviceSwitch = optimizeBtDeviceSwitch() && d.mNewDevice != null && d.mPreviousDevice != null`
(`AudioDeviceBroker.java:1015`).

When the BT speaker re-attaches on wake from deep sleep,
`optimize_bt_device_switch` causes the audio framework to **skip** the
`checkSendBecomingNoisyIntentInt(...)` delay. Becoming-noisy broadcast
isn't sent, media apps don't pause, audio path renegotiates while
playback is in flight. Maps directly to "cracky audio for some seconds,
then recovers."

LOS22 (A15) doesn't have this flag in any release config — every BT
device-active event went through the proper delayed path, no rush.

## How to install

This swaps just `/system/framework/services.jar` (audio service is
inside it). Three install paths in increasing complexity:

### Path A — full fastboot flash from build host (cleanest)

The build host has a fresh `system.img` containing the patched
`services.jar` baked in. Easier than replicating dex2oat on device.

```bash
adb -s 5000194162 reboot bootloader
# on build host:
cd /data4/LOS23-build && echo "user" | sudo -S bash flash-imgs-all.sh
```

### Path B — services.jar swap (fast, requires writable /system)

Backup, push, fix permissions, drop dexopt artefacts so system regenerates them on next boot:

```bash
adb shell 'mount -o rw,remount /'
adb shell 'cp /system/framework/services.jar /sdcard/claude/bt-lag/services.jar.before-audioflags-$(date +%Y%m%d-%H%M%S)'
adb push services_audio_flags_disabled.jar /system/framework/services.jar
adb shell 'chown root:root /system/framework/services.jar && chmod 644 /system/framework/services.jar'
adb shell 'restorecon /system/framework/services.jar'
# Drop pre-compiled odex/vdex so system_server re-dexopts on boot
adb shell 'rm -f /system/framework/oat/arm64/services.{odex,vdex,art}'
adb reboot
```

First boot after this is slower (~30s extra) for re-dexopt. If the
device boots fine, the patched services.jar is live.

If the device bootloops, fall back to Path A or Path C.

### Path C — Magisk module (overlay, doesn't touch real /system)

Build a Magisk module with the patched services.jar at
`module/system/framework/services.jar`. Requires Magisk root.
Build session can produce one if needed.

## Verification on device

Once running:

```bash
adb shell 'cmd device_config get com.android.media.audio optimize_bt_device_switch'
adb shell 'cmd device_config get com.android.media.audio as_device_connection_failure'
```

Both should be `false`. AudioService dump should also show:

```bash
adb shell 'dumpsys audio | grep -E "optimizeBtDeviceSwitch|asDeviceConnectionFailure"'
# expected:
#   com.android.media.audio.optimizeBtDeviceSwitch: false
#   com.android.media.audio.as_device_connection_failure: false
```

## Test

Same protocol as round 1: lock + ≥30s deep sleep + unlock with BT
music playing. Listen for the audible glitch + cracky audio.

Acceptance: lag absent / glitch single-frame and inaudible.

If it works → next step: re-enable one of the two flags individually
to find the minimal-disable set.

If it doesn't work → the audio framework flag space is also ruled out;
remaining suspects are AudioFlinger threading (PlaybackThread / Tracks),
audio HAL service (`hardware/interfaces/bluetooth/audio/aidl/default/`),
or a deeper Mainline BT module rebuild.

## Independent confounder discovered today

`device/blackberry/sdm660-common` commit `2130948` (May 2) enabled A2DP
offload to QDSP6 by flipping three vendor props:
- `persist.bluetooth.a2dp_offload.disabled    true → false`
- `ro.bluetooth.a2dp_offload.supported        false → true`
- `vendor.audio.feature.a2dp_offload.enable   false → true`

This may cause an audio quality regression independent of the BT-lag
investigation, since BBRY vendor blobs use legacy
`com.qualcomm.qti.bluetooth_audio@1.0` HIDL — the QDSP6 offload path
may not have proper ACDB calibration for athena. **Not reverted in
this build.** If sound quality on the new APEX is worse than baseline,
the offload commit is the suspect to revert next.
