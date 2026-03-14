# LineageOS 22.2 for BlackBerry KEY2 (athena)

Changes relative to [XDA ROM by Igor (botched)](https://xdaforums.com/t/rom-15-unofficial-beta-athena-lineageos-22-2-for-blackberry-key2.4777062/)  
Device-tree: https://github.com/FumoEnterprises/
Mostly developed by Igor (BotchedRPR)

## Custom-rom by krab-ubica-v1.0
### based on [device-tree](https://github.com/FumoEnterprises/) at date 09.03.2026  

## Changes to ROM at 01.03.2026 

- SELinux enforcing
- Encryption
- DT2W, button fixes on Synaptics and Focal
- Volume curve fixes
- RCS HAL crash fix
- Bluetooth MAC address loader
- Touch keypad now usable
- Keyboard working in all apps
- Wi-Fi hotspot fixes
- Init fixes
- ExFAT support
- [Lokker](https://github.com/tim-ecoder/Lokker) — analogue of BlackBerry Locker
- ALT+ in AOSP works
- Keypad touch paging mode (same as Oreo)
- BT audio routing to headset and car
- GPS stability
- LTE stability
- IME switching preference (Sym → Alt+Enter, etc.)
- Quick Settings tile to toggle keypad_touch
- Signed with krab-ubica keys

## Patches applied

/patches

## Notes

Playing it open: all sources are open, applied patches are published.

> Some disagreements arose regarding improvements — a number of Igor's commits had to be reverted
> to preserve the stability of [correct](https://github.com/tim-ecoder/KeyoneKB) keyboard behavior.

When BlackBerry KEY2 (athena) receives official LineageOS support, some keyboard features will be
problematic due to LineageOS architectural requirements — so convenience extras will only be
available in custom ROMs like this one.
