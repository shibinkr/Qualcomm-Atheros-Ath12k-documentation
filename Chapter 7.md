# Chapter 7: Power Management

Power Management (PM) in the **ath12k** driver involves coordination across the **mac80211** subsystem, the **ath12k** driver, and the firmware/hardware to reduce power consumption while maintaining connectivity and responsiveness. This is especially important for client devices (STAs) like phones or laptops.

## 7.1 Power Management Levels

| Level              | Description                                                                                   |
|-------------------|-----------------------------------------------------------------------------------------------|
| Runtime PM         | Driver/hardware powers down during idle times.                                               |
| mac80211 PSM       | STA tells AP it's going into power save mode (802.11 Power Save Mode).                       |
| WMI Powersave Cmds | Driver instructs firmware to enter or exit powersave.                                         |
| WoWLAN             | Wake on Wireless — wake host from suspend on specific wireless events.                        |
| Target Wake Time (TWT) | Wi-Fi 6 feature: negotiate exact wakeup time with AP to optimize sleep.                    |

## 7.2 Power Save Flow (STA)

- Userspace (e.g., NetworkManager or wpa_supplicant) enables power save (PS).
- **mac80211** notifies the driver via `ops->config`.
- **ath12k** sets firmware PS mode via WMI.
