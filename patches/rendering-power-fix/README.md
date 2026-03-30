Rendering Power Fix — Reduce battery drain during heavy GPU load

1. Removed EXPENSIVE_RENDERING GPU lock (powerhint.json)
   - Was locking GPU min+max to 647 MHz, preventing downclock

2. GPU thermal threshold 70→80°C, skin 50→60°C (thermal-engine.conf)
   - Less aggressive throttling reduces oscillating boost/throttle cycles

3. Restored use_sched_load=1, use_migration_notif=1 (init.qcom.post_boot.sh)
   - Match retail — scheduler-driven governor for better freq decisions

4. max_freq_hysteresis=79000 both clusters (init.qcom.post_boot.sh)
   - Tuned from 59000

5. CABL disabled (system.prop)
   - Not functional with PIE display blobs — set to 0 to avoid dpps overhead

Files:
  device-sdm660.patch: powerhint.json + init.qcom.post_boot.sh + system.prop
  vendor-thermal.patch: thermal-engine.conf
