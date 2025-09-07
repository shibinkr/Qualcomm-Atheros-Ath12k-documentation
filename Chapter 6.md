# Chapter 6: Scan, Authentication & Association

## 1. Scan (Active or Passive)

**Purpose:** Discover nearby access points (APs).

### Sequence & Details

- User triggers scan (`iw dev wlan0 scan`) or automatic background scan.
- `cfg80211` handles request → passes to `mac80211`.
- `mac80211_ops::hw_scan()` or `sw_scan()` is called.
- In **ath12k**:
  - `ath12k_mac_op_hw_scan()` sets up scan parameters.
  - Firmware (via WMI) executes the scan (sending probe requests if active).
  - Scan results come back via `ath12k_wmi_event_scan_*` handlers.
  - Results are passed up to `cfg80211` → userspace (e.g., `iw` or NetworkManager).

```code 
Userspace (iw / NetworkManager)
└─ cfg80211_scan()
   └─ Triggers scan request to mac80211
      └─ mac80211 (ops->hw_scan)
         └─ Calls ath12k_mac_op_hw_scan()
            ├─ Select appropriate radio (AR)
            ├─ Assign or create link/VDEV
            ├─ Allocate and populate scan arguments
            └─ Send WMI_START_SCAN command to firmware
               └─ Firmware
                  ├─ Performs scan:
                  │  ├─ Active: Sends Probe Requests
                  │  └─ Passive: Listens for Beacons
                  └─ Scan Results
                     ├─ Received by firmware
                     ├─ Reported to driver via WMI_SCAN_EVENT
                     ├─ Reported back to cfg80211
                     └─ Eventually reaches userspace (iw, NM)
```

## 2. Authentication

**Purpose:** Initiate connection to AP with basic authentication.

**Sequence:**
```text
Userspace (e.g., wpa_supplicant, NetworkManager)
└─ Requests connection to an AP
   └─ Calls cfg80211_connect()
      └─ Triggers mac80211 connect flow
         └─ mac80211 (ops->connect → ath12k_mac_op_connect())
            └─ Assigns or ensures VDEV is created for STA
            └─ Populates WMI_PEER_CREATE + VDEV_START (if needed)
            └─ Sends WMI_PEER_AUTH command to firmware
               ├─ Includes auth algorithm (e.g., Open, Shared Key)
               ├─ Specifies peer MAC (the AP)
               └─ Firmware processes authentication
                  ├─ Sends frame over the air (Auth Request)
                  └─ Waits for AP’s Auth Response
                     ├─ If successful:
                     │  └─ WMI_PEER_AUTH_EVENT received
                     │     ├─ Indicates authentication success
                     │     └─ Driver notifies mac80211
                     │        └─ Triggers association phase
                     └─ If failed:
                        └─ WMI_PEER_AUTH_EVENT indicates failure
                           └─ mac80211/cfg80211 notified of failure

```

**Details:**
- `cfg80211_connect()` → `mac80211_ops::connect()`.
- In `ath12k`:
  - `ath12k_mac_op_connect()` prepares the connection request.
  - Sends WMI command to firmware for authentication.
  - Firmware transmits Auth frame over air.
  - Receives Auth response from AP.
  - Confirmation received back via `ath12k_wmi_event_peer_sta_auth`.

## 3. Association

**Purpose:** Complete negotiation and join AP.

**Sequence:**
- Association request sent by firmware
- AP responds with Association Response
- Response forwarded up to `cfg80211`

**Details:**
- After authentication success, association request is initiated by firmware.
- WMI commands used (e.g., `wmi_peer_assoc_cmd`).
- Association response (status, AID, capabilities) comes back.
- Indicated to `mac80211` via `ieee80211_connection_result()`.
- `cfg80211` notifies userspace: connection is established.
