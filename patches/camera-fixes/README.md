# Camera Fixes

Four patches fixing camera issues on BlackBerry Key2 (SDM660, OREO blobs on LOS22).

## Patches

### kernel-drop-reconfig.patch
**ISP drop_reconfig cascade fix.** MCT never clears `drop_reconfig` flag via mmap'd page,
causing infinite OUTPUT_ERROR cascade on BLOB streams → ISP unrecoverable state.
Fix: auto-clear in EPOCH IRQ handler + rate-limit error log.
- File: `kernel/.../isp/msm_isp_axi_util.c`

### kernel-flash-torch.patch
**Dual-tone flash/torch fixes.** Three issues:
1. Flash init blocked by enum zero-match (reordered enum)
2. Single yellow LED — HAL only sets flash_current[0], mirrored to [1]
3. Torch yellow tint — HAL sends low current, kernel now uses DT default (300mA) as floor
- Files: `kernel/.../sensor/flash/msm_flash.c`, `msm_flash.h`

### kernel-camif-disable.patch
**CAMIF FSM stall on rapid stream reconfigure.** When HAL reconfigures streams mid-startup,
`DISABLE_CAMIF` writes 0x0 leaving FSM in bad state. Subsequent `ENABLE_CAMIF` fails silently
→ zero SOF for 5s → HW_ERROR → black screen.
Fix: always use `DISABLE_CAMIF_IMMEDIATELY` (0x6 = STOP+CLEAR) which resets FSM to idle.
- File: `kernel/.../isp/msm_isp_axi_util.c`

### device-iface-bundle-mask.patch
**iface_modules bundle mask premature clear.** During rapid app switch, HAL reconfigure causes
partial streamoff that clears bundle mask → all frames return buf_err → black screen.
Fix: binary patch at 0x4c6ae (cbnz→b.n) to skip mask clear in `iface_streamoff`.
- File: `device/blackberry/athena/extract-files.py`
