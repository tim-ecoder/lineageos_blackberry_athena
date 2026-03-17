CPU Governor Fix — Idle Frequency Scaling
=========================================

Problem: Little cluster (CPU0-3) stuck at max frequency (1843200 Hz) while idle.
Caused ~15°C extra heat and significant battery drain.

Root cause: use_sched_load=1 on 4.4 kernel causes interactive governor to
overestimate CPU load via scheduler load tracking, preventing frequency scaling.

Fix in init.qcom.post_boot.sh:
  Little cluster (CPU0):
    use_sched_load: 1 -> 0 (disable broken scheduler load estimation)
    use_migration_notif: 1 -> 0 (disable migration notifications keeping freq high)
    max_freq_hysteresis: 0 -> 59000 (add 59ms hysteresis, matches big cluster)

  Big cluster (CPU4):
    use_sched_load: 1 -> 0
    use_migration_notif: 1 -> 0

Results:
  CPU idle freq: 1843200 (stuck) -> 1401600-1747200 (scaling)
  CPU temp: 50-56°C -> 37-38°C (-15°C!)
  GPU temp: 46°C -> 35°C
  Skin temp: 45-46°C -> 34-35°C
