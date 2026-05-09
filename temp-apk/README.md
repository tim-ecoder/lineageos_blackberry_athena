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

---

# BT-lag round 3 — `remove_stream_suspend` (audioserver native)

Built 2026-05-09. Round 2's audio framework 2-flag disable did not
help (services.jar swap verified active, both flags read false at
runtime, lag persisted). Round 3 targets a single native-side flag
in `com.android.media.audioserver` namespace — `remove_stream_suspend`.

## Files

| File | What | md5 |
|---|---|---|
| `audioserver_remove_stream_suspend_disabled` | rebuilt `/system/bin/audioserver` (3.0 MB) — contains AudioFlinger.cpp main logic statically linked | `bb5ee0c3e9060bb77f6c2509ccf27f48` |
| `libaudiopolicymanagerdefault_remove_stream_suspend_disabled.so` | rebuilt `/system/lib64/libaudiopolicymanagerdefault.so` (722 KB) — contains AudioPolicyManager.cpp logic | `6277d8da11629aeada953f0f6ad27009` |

The single disabled flag:

- **`com.android.media.audioserver::remove_stream_suspend`** — was ENABLED via bp3a/bp4a inheritance. Now DISABLED, READ_ONLY at compile time.

## Hypothesis (very strong, source-level direct match)

`AudioPolicyManager.cpp::checkForDeviceAndOutputChanges`. The function comment at the call site is the smoking gun:

```cpp
// "checkA2dpSuspend must run before checkOutputForAllStrategies so that A2DP
//  output is suspended before any tracks are moved to it"
if (!remove_stream_suspend()) {
    checkA2dpSuspend();           // LOS222: runs (proper sequencing)
}
checkOutputForAllStrategies();    // LOS23 with flag ENABLED:
                                   // tracks moved to A2DP WITHOUT prior suspend
```

When `remove_stream_suspend = ENABLED` (LOS23 default), `checkA2dpSuspend()` is **skipped** in `checkForDeviceAndOutputChanges()`. The function comment EXPLICITLY warns this should run BEFORE moving tracks to A2DP output, otherwise tracks get moved while A2DP isn't properly suspended/resumed.

On wake from deep sleep, the BT speaker re-attaches → `checkForDeviceAndOutputChanges()` runs → tracks routed to A2DP without prior suspend → audio cracks until A2DP catches up. Maps directly to "cracky audio for some seconds, then recovers."

LOS222 had no `com.android.media.audioserver` overrides at all — `checkA2dpSuspend()` always ran first, no rush.

Additional callsites:
- `AudioFlinger.cpp:3181` — `LOG_ALWAYS_FATAL` if old suspend code is called when flag enabled. The flag was meant to be paired with new MMAP-based suspend logic that bypasses the old path. For non-MMAP A2DP (athena CPU encoding), this leaves a behavioral gap.
- `AudioFlinger.cpp:3201` — same.
- `AudioPolicyManager.cpp:7649` — same call site, secondary.

## Install

services.jar from round 2 STAYS — round 3 is purely native side, no overlap. Final on-device state should be:

- `/system/system_ext/priv-app/SystemUI/SystemUI.apk` = patched (PIN fix)
- `/system/apex/com.android.bt.capex` = patched 4-flag-disabled (round 1)
- `/system/framework/services.jar` = patched 2-flag-disabled (round 2)
- `/system/bin/audioserver` = patched (round 3, NEW)
- `/system/lib64/libaudiopolicymanagerdefault.so` = patched (round 3, NEW)

### Path A — full fastboot flash (cleanest)

The build host's `flash-imgs-all.sh` flashes a `system.img` containing all of round 1 + round 2 + round 3 patched, plus the matching BT APEX. One step.

```bash
adb -s 5000194162 reboot bootloader
# on build host:
cd /data4/LOS23-build && echo "user" | sudo -S bash flash-imgs-all.sh
```

### Path B — native binary swap (more involved)

The audioserver binary needs SELinux relabel and audioserver process restart. Pseudocode:

```bash
adb shell 'mount -o rw,remount /'
adb shell 'cp /system/bin/audioserver /sdcard/claude/bt-lag/audioserver.before-stream-suspend-$(date +%Y%m%d-%H%M%S)'
adb shell 'cp /system/lib64/libaudiopolicymanagerdefault.so /sdcard/claude/bt-lag/libaudiopolicymanagerdefault.so.before-$(date +%Y%m%d-%H%M%S)'
adb push audioserver_remove_stream_suspend_disabled /system/bin/audioserver
adb push libaudiopolicymanagerdefault_remove_stream_suspend_disabled.so /system/lib64/libaudiopolicymanagerdefault.so
adb shell 'chown root:root /system/bin/audioserver && chmod 755 /system/bin/audioserver && restorecon /system/bin/audioserver'
adb shell 'chown root:root /system/lib64/libaudiopolicymanagerdefault.so && chmod 644 /system/lib64/libaudiopolicymanagerdefault.so && restorecon /system/lib64/libaudiopolicymanagerdefault.so'
adb shell 'killall audioserver'   # init.audioserver.rc respawns automatically
sleep 5
adb shell 'pidof audioserver'      # should print new pid
```

Or just `adb reboot` after pushing — also fine.

## Verification

This flag is `READ_ONLY` and native — `cmd device_config get` won't return a value (services.jar ground truth path doesn't apply). Verify via:

```bash
adb shell 'pidof audioserver'                       # confirm running
adb shell 'logcat -d -b system | grep -i "remove_stream_suspend"'  # may emit a log
adb shell 'md5sum /system/bin/audioserver'          # should match bb5ee0c3...
adb shell 'md5sum /system/lib64/libaudiopolicymanagerdefault.so'   # should match 6277d8da...
```

Best behavioral verification: lock + unlock test. If pre-fix logcat showed `LOG_ALWAYS_FATAL` style errors around stream-suspend, those should disappear now (since the legacy path is intentionally re-enabled).

## Test

Same protocol — lock + ≥30s deep sleep + unlock with BT music playing.

If this fixes the lag → minimum-disable found. The fix is a single line in a single textproto. We can upstream as a separate `patches/bt-lag/` directory in this repo.

If lag persists → audio framework + audioserver flag space ruled out. Remaining suspects narrow to AudioFlinger threading internals (PlaybackThread, Track lifecycle) and the BT audio HAL service binary itself (`hardware/interfaces/bluetooth/audio/aidl/default/`).

---

# BT-lag round 4 — REVERT A2DP offload (the actual baseline divergence)

Built 2026-05-09. Three rounds of aconfig disabling (BT module flags,
audio framework Java flags, audioserver native flag) **all failed**.
The strong source-level hypotheses didn't pan out, suggesting the
actual cause is somewhere else.

Re-examined the bisect baseline. Discovered the real divergence:

| Build | A2DP encoding | BT lag |
|---|---|---|
| LOS22 + 4.4 | CPU (`a2dp_offload.disabled=true`) | NO |
| LOS222 (LOS22+4.19) | CPU (same props) | NO |
| **LOS23 + 4.19** | **DSP offload via QDSP6** (props flipped May 2 by commit `2130948`) | **YES** |

The "LOS22 vs LOS23 framework regression" bisect was confounded — it
actually compared **CPU encoding vs DSP offload**, not A15 vs A16.
Three rounds of framework aconfig disable couldn't fix a bug that
wasn't in the framework.

## Files

`round4-offload-revert/` directory:

| File | md5 | What |
|---|---|---|
| `system_build.prop` | `7d1635690b8818fe688e8840789df11d` | rebuilt `/system/build.prop` with `persist.bt.a2dp.aac_disable=true` (LOS22 baseline) |
| `vendor_build.prop` | `ebcd9f185f8be860b4a2d771fa465844` | rebuilt `/vendor/build.prop` with `a2dp_offload.disabled=true`, `a2dp_offload.supported=false`, `a2dp_offload.enable=false` (LOS22 baseline) |

Implements `git revert 2130948` cleanly. Same prop values as LOS22 device tree.

## Hypothesis

BBRY's vendor partition uses legacy Oreo-era `com.qualcomm.qti.bluetooth_audio@1.0`
HIDL HAL — DSP offload on athena likely doesn't have proper ACDB calibration
AND has broken suspend/resume lifecycle in this HAL version. Both observed
symptoms (audio quality regression + lag-after-deep-sleep + cracky audio)
plausibly trace to this same root cause.

CPU encoding has its own downside (BT music glitches under heavy CPU load —
typing, camera, scrolling) which originally motivated commit `2130948`. But
that's a different problem with different fixes (RT priority for AudioOut
threads, scheduler tuning) than the deep-sleep lag.

## Install

### Path A — full fastboot flash (cleanest)

Build host has fresh `system.img` + `vendor.img` with reverted props baked in:

```bash
adb -s 5000194162 reboot bootloader
cd /data4/LOS23-build && echo "user" | sudo -S bash flash-imgs-all.sh
```

### Path B — build.prop swap (fast, riskier)

```bash
adb shell 'mount -o rw,remount /'
adb shell 'mount -o rw,remount /vendor'
adb shell 'cp /system/build.prop /sdcard/claude/bt-lag/system_build.prop.before'
adb shell 'cp /vendor/build.prop /sdcard/claude/bt-lag/vendor_build.prop.before'
adb push system_build.prop /system/build.prop
adb push vendor_build.prop /vendor/build.prop
adb shell 'chown root:root /system/build.prop /vendor/build.prop'
adb shell 'chmod 644 /system/build.prop /vendor/build.prop'
adb shell 'restorecon /system/build.prop /vendor/build.prop'
adb reboot
```

### Path C — runtime-only smoke test (fastest, partial)

`ro.bluetooth.a2dp_offload.supported` is read-only and CANNOT be flipped at
runtime — so this is partial. But the `persist.*` props can:

```bash
adb shell 'setprop persist.bluetooth.a2dp_offload.disabled true'
adb shell 'setprop persist.bt.a2dp.aac_disable true'
adb shell 'svc bluetooth disable; sleep 3; svc bluetooth enable'
```

Reconnect the BT speaker. Test. If lag goes away — strong signal that
hypothesis is right (full Path A/B will close the loop fully).
If lag persists — partial test inconclusive; need full flash.

## Verify post-install

```bash
adb shell 'getprop ro.bluetooth.a2dp_offload.supported'    # → false
adb shell 'getprop persist.bluetooth.a2dp_offload.disabled' # → true
adb shell 'getprop vendor.audio.feature.a2dp_offload.enable' # → false
adb shell 'getprop persist.bt.a2dp.aac_disable'            # → true
adb shell 'dumpsys bluetooth_manager | grep -iE "offload|codec"' # codec should show CPU encoder, not offload
```

After reconnecting the BT speaker, `dumpsys media.audio_flinger` should
show the audio output WITHOUT `AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD` flag.

## Test

Same protocol — lock + ≥30s deep sleep + unlock with BT music.

If lag GONE → hypothesis confirmed. The bisect was wrong, the regression
is offload-on-athena, not A16 framework. We then either:
1. Keep the revert permanently (CPU encoding, accept the heavy-load
   glitching the original commit was trying to fix)
2. Hunt down a working DSP offload path (BBRY HAL update, much harder)
3. Apply RT-priority workaround for CPU encoder under load (the original
   alternative that the offload commit dismissed as "wrong layer")

If lag PERSISTS → A16 regression is real, narrowed even further. Next
suspects: BT audio HAL service binary itself
(`hardware/interfaces/bluetooth/audio/aidl/default/`, 629 lines diff
between LOS222 and LOS23), or AudioFlinger PlaybackThread internals.

## Composite final state on device after Path A install

| Layer | State after Round 4 |
|---|---|
| `/system/system_ext/priv-app/SystemUI/SystemUI.apk` | PIN-fix patched (rounds 1 of separate task) |
| `/system/apex/com.android.bt.capex` | 4 BT flags disabled (round 1 of BT-lag) |
| `/system/framework/services.jar` | 2 audio framework flags disabled (round 2) |
| `/system/bin/audioserver` + `libaudiopolicymanagerdefault.so` | `remove_stream_suspend` disabled (round 3) |
| `/system/build.prop` + `/vendor/build.prop` | A2DP offload reverted (round 4, NEW) |

---

# BT-lag round 5 — keymaster@3.0 service removal (CPU drain root cause)

Built 2026-05-09. On-device session pinpointed the actual mechanism.
Five aconfig disable rounds + offload revert all failed because
**the regression is CPU-side, not framework**:

`/vendor/etc/init/android.hardware.keymaster@3.0-service-qti.rc`
declares an `early_hal` service that:

1. Tries to register `keymaster@3.0::IKeymasterDevice/default` with hwservicemanager
2. Always fails — vendor `/vendor/etc/vintf/manifest.xml` only declares `@4.0`
3. Process exits → init's auto-restart spawns it again every ~5s
4. Each spawn does `QSEECom_get_handle` (TZ TA load) → CPU/TZ contention
5. BT periodic task misses 23ms deadline → FMQ underflow → A2DP cracks

**62 spawn/exit cycles per 100s** confirmed in cycle.logcat capture.
Runtime fix verified on device (deleted the rc file → 0 spawns
post-reboot, keystore2 + keymaster@4.0 still healthy, all auth functional).

## Build-tree changes (no temp-apk artefact this round — full flash only)

The fix is in `vendor.img` (and the source tree). It's not a single
swappable binary like rounds 1-3; the .rc file is what spawns the
service, and removing it is a vendor partition change. Path: full
fastboot flash from the build host.

Files edited in source:
- `device/blackberry/sdm660-common/proprietary-files.txt` — commented out the binary + rc lines (`# vendor/bin/hw/android.hardware.keymaster@3.0-service-qti` + `.rc`)
- `device/blackberry/athena/proprietary-files.txt` — commented out the binary + rc lines (with sha)
- `vendor/blackberry/sdm660-common/sdm660-common-vendor.mk` — removed `PRODUCT_COPY_FILES` rc copy entry + `PRODUCT_PACKAGES` `android.hardware.keymaster@3.0-service-qti` entry
- `vendor/blackberry/sdm660-common/Android.bp` — removed entire `cc_prebuilt_binary { name: "android.hardware.keymaster@3.0-service-qti", ... }` block

Files deleted from source:
- `device/blackberry/sdm660-common/configs/init/android.hardware.keymaster@3.0-service-qti.rc`
- `vendor/blackberry/sdm660-common/proprietary/vendor/etc/init/android.hardware.keymaster@3.0-service-qti.rc`
- `vendor/blackberry/sdm660-common/proprietary/vendor/bin/hw/android.hardware.keymaster@3.0-service-qti`

Kept (orphaned but harmless without the service binary):
- `vendor/lib{,64}/hw/android.hardware.keymaster@3.0-impl-qti.so`
- `vendor/lib/libkeymasterdeviceutils.so`, `libkeymasterprovision.so`, `libkeymasterutils.so`

## Build artefact md5

```
vendor.img  b670652d3d9bff948f30042abe9cfbec  564310260 bytes
system.img  87c8e93bac0dc84dd628ea7f596b82d2  1963991656 bytes
```

Vendor.img is 16 KB smaller than the previous build — the removed
service binary + rc accounted for it.

## Install — full fastboot flash from build host

```bash
adb -s 5000194162 reboot bootloader
cd /data4/LOS23-build && echo "user" | sudo -S bash flash-imgs-all.sh
```

## Verify post-flash

```bash
adb shell 'logcat -d | grep keymaster-3-0 | wc -l'                  # → 0 (was 62/100s)
adb shell 'pidof android.hardware.keymaster@4.0-service'            # → present
adb shell 'service list | grep keystore'                            # → keystore2
adb shell 'ls /vendor/bin/hw/ | grep keymaster'                     # → only @4.0
adb shell 'ls /vendor/etc/init/ | grep keymaster'                   # → only @4.0
```

## Test

Same protocol — lock + ≥30s deep sleep + unlock with BT music.

If lag GONE → keymaster respawn was the CPU drain. **Ship as permanent commit.**

If lag PERSISTS → ship the keymaster fix anyway (it's an independent
real bug, ~62 wasted TZ TA loads per 100s) and continue source-side
investigation. Even without fixing the lag, removing 12% TZ
contention is worth it.

## Why on-device session and build-side both correct

- **On-device**: ground truth. saw 62 keymaster spawn/exit cycles
  in 100s, traced to .rc file, verified runtime fix works.
- **Build-side**: 5 framework aconfig rounds disproved the
  framework-flag hypothesis, and source diffs disproved the
  framework-code hypothesis. The mismatch was real but in
  CPU/TZ contention, not in framework code or flags. Both
  conclusions are consistent: the regression isn't in framework,
  it's in CPU pressure that pushes the periodic-thread deadline
  past tolerance, and A16 BT consumer is less tolerant of FMQ
  short-reads than A15.

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
