GPS XTRA Fix — FINAL WORKING (folder3)
=======================================

Status: XTRA downloads within 60s, injects without errors, GPS fixes outdoors.
Tested: clean boot, XTRA cache cleared, fresh download + inject confirmed.

Root causes fixed:
1. getExtensionXtra() returned nullptr → Android 15 disabled AGPS+XTRA pipeline
2. LOC_HIDL_VERSION mismatch (4.2 vs 4.0) → vendor enhanced service didn't load
3. va_odm.support not set → vendorEnhanced=false
4. XTRA server URLs not configured → xtra-daemon had nothing to download
5. XTRA_INTERVAL not set → download delayed by hours (now 60s)
6. SUPL_HOST caused 500m cell-tower jumps
7. xtra-daemon binary patched to read config from /vendor/etc/ (read-only, always available)
8. XTRA v3 (xtra3grc.bin) instead of v2 — v2 caused intermittent inject errors

Changes:

SOURCE (device/blackberry/sdm660-common/):
  gps/android/2.1/GnssXtra.h + GnssXtra.cpp — IGnssXtra stub (enables framework PSDS pipeline)
  gps/android/2.1/Gnss.h — getExtensionXtra() returns new GnssXtra()
  gps/android/2.1/Android.bp — GnssXtra.cpp added, LOC_HIDL_VERSION="4.0"
  gps/gnss/GnssAdapter.h + .cpp — requestXtraData() override
  gps/gnss/XtraSystemStatusObserver.cpp — respondStatus + connectionActiveEvent on connectivity
  gps/etc/gps.conf — XTRA_SERVER xtra3grc.bin, XTRA_INTERVAL=60, SUPL removed, RF_LOSS, retail alignment
  gps/etc/izat.conf — NLP_MODE=4, GTP_CELL/WIFI=BASIC, SAP=DISABLED, ODCPI=BASIC
  vendor.prop — ro.vendor.qti.va_odm.support=1
  init/init.qcom.rc — gps.prop copy
  sepolicy/vendor/init.te — location_data_file write for init

VENDOR BLOBS (vendor/blackberry/sdm660-common/):
  bin/xtra-daemon — Nokia donor, binary patched: path → /vendor/etc/xtra_single_boot.conf
  lib64/libloc_pla.so + libloc_stub.so — OREO deps (for future OREO xtra-daemon if needed)
  etc/izat.conf — xtra-daemon, lowi-server, xtwifi-* ENABLED
  etc/xtra_single_boot.conf — XTRA_SERVER URLs (xtra3grc.bin)
  etc/gps.prop — empty (prevents "wrong format" error)
  sdm660-common-vendor.mk — added gps.prop, xtra_single_boot.conf, libloc_pla, libloc_stub
  Android.bp — cc_prebuilt_library_shared for libloc_pla, libloc_stub
