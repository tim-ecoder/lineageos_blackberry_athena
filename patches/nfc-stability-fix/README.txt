NFC Stability Fix (updated)
===========================

Target files:
  device/blackberry/sdm660-common/configs/nfc/libnfc-nxp.conf
  device/blackberry/sdm660-common/configs/nfc/libnfc-nci.conf

Changes in libnfc-nxp.conf:
  1. NXP_NFC_CHIP: keep 0x08 (PN80T) — MUST match actual hardware!
     Was wrongly changed to 0x07 (PN553) which caused 211 "Empty packet received"
     errors per boot + "unknown opcode:0x23" spam every 1.7 seconds.
     Chip ID controls HAL communication protocol — must match hardware regardless of eSE.
  2. NXP_DEFAULT_SE: 0x07 -> 0x02 (UICC only, no eSE)
  3. DEFAULT_OFFHOST_ROUTE: 0x01 -> 0x02 (route to UICC instead of non-existent eSE)
  4. DEFAULT_NFCF_ROUTE: 0x01 -> 0x02 (Felica route to UICC instead of eSE)
  5. DEFAULT_SYS_CODE_ROUTE: 0xC0 -> 0x00 (system code route to host, no eSE)
  6. NXP_DEFAULT_NFCEE_DISC_TIMEOUT: 0 -> 20 (restore proper discovery timeout)
  7. NXP_DEFAULT_NFCEE_TIMEOUT: 0 -> 20 (restore proper NFCEE timeout)
  8. NXP_ESE_POWER_DH_CONTROL: 1 -> 0 (disable eSE power control, no eSE present)
  9. DEVICE_HOST_WHITE_LIST: {C0, 02} -> {02} (remove eSE C0 from HCI host list)
  10. All NXPLOG levels: 0x03 (DEBUG) -> 0x01 (ERROR only)

Changes in libnfc-nci.conf:
  1. APPL_TRACE_LEVEL: 0xFF -> 0x01 (ERROR only)
  2. PROTOCOL_TRACE_LEVEL: 0xFFFFFFFF -> 0x00000001 (ERROR only)
  3. NFC_DEBUG_ENABLED: 0x01 -> 0x00 (disable debug)

Key: eSE unusable (GPIO 31 conflict with touchpad IRQ), but NXP_NFC_CHIP must
stay 0x08 (PN80T) to match actual hardware. eSE disabled via routing only.
