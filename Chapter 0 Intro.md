# Ath12k
The Qualcomm Atheros Wi-Fi 7 driver included in Linux

---

## Introduction

### 1. Module Load & Driver Registration
- The driver is registered during kernel build via `module_init(ath12k_register)`,  
  which calls `ath12k_pci_register()` in **pci.c**.  
- This registers a PCI driver (`struct pci_driver ath12k_pci_driver`) with **probe** and **remove** callbacks for devices that match Atheros Wi-Fi 7 hardware.

---

### 2. PCI Probe (`ath12k_pci_probe()`)
Triggered when matching hardware is detected.  

**Steps:**
1. Enables the PCI device and sets DMA mask.  
2. Requests PCI regions and maps BAR0 I/O registers.  
3. Allocates MSI/MSI-X IRQs (typically 16 vectors).  
4. Allocates and initializes `struct ath12k` via `ath12k_core_create()`.  

---

### 3. Core Initialization (`ath12k_core_create()` → `ath12k_core_start()`)
- **`ath12k_core_create()`** (in **core.c**):
  - Initializes data structures (`struct ath12k`).  
  - Sets up mac80211 wiphy (`ath12k_wiphy_init()`).  
  - Initializes workqueues and allocates TX/RX rings.  

- **`ath12k_core_start()`**:
  - Downloads firmware via **fw.c** routines.  
  - Uses WMI commands to configure firmware and start services.  
  - Registers wiphy with mac80211 via `wiphy_register()`.  
  - Allocates IRQs and starts RX buffer refilling.  

---

### 4. TX Path
- **Flow:**  
  `mac80211 → ath12k_mac_op_tx()` (via `struct ieee80211_ops` in **mac.c**)  
  → Maps packet into TX ring using descriptor creation (**hw.c/hal.c**)  
  → Doorbell poke to notify firmware/hardware.  

- **IRQ Handler (irq.c):**  
  On TX completion, frees skb and notifies mac80211 (`ieee80211_tx_status()`).  

---

### 5. RX Path
- RX buffers are pre-posted into RX rings.  
- On packet arrival:  
  - `ath12k_msi_bottom_half()` processes the interrupt.  
  - Calls `ath12k_process_rx()`, which parses descriptors via **HAL** (`hal_rx_parse()`).  
  - Builds SKB, populates metadata (rate, RSSI, flags).  
  - Submits packet to mac80211 via `ieee80211_rx_irqsafe()`.  

---

### 6. Scan, Authentication & Association
Triggered by **mac80211** via `struct ieee80211_ops`:  
- **Scan:** `ath12k_mac_op_scan()` issues WMI scan-start and returns results.  
- **Auth/Assoc:** `ath12k_mac_op_sta_add()` / `assoc()` → WMI commands → Firmware responses → mac80211 events.  

---

### 7. Power Management
- Supports **PCI ASPM**, **DTIM-based Power Save**, and **runtime PM**.  
- Configuration via **Kconfig** + HAL routines in **hal.c/PCI** portions.  

---

### 8. Error Handling & Firmware Recovery
- On firmware timeout/hang or missing `WMI_READY`, driver calls `ath12k_core_recover()`:  
  - Stops TX/RX, resets firmware, re-downloads firmware.  
  - Re-initializes descriptors, reapplies mac80211 state (scan/auth/etc).  
- Enables robust recovery when firmware misbehaves.  

---

### 9. Device Removal (`ath12k_pci_remove()`)
Called on module unload or hot-unplug.  

**Steps:**
1. `ath12k_core_stop()` — stops mac80211, firmware services, and queues.  
2. `ath12k_core_destroy()` — frees rings, workqueues, HW state.  
3. Releases IRQs and unmaps I/O registers.  
4. Frees `struct ath12k` and unregisters PCI device.  

---

## 📊 Summary Table

| Stage               | Key Functions                        | Files                    |
|----------------------|--------------------------------------|--------------------------|
| Driver Registration  | `module_init()`, `ath12k_register()` | pci.c, core.c            |
| PCI Probe            | `ath12k_pci_probe()`                 | pci.c                    |
| Firmware Init        | `ath12k_core_start()`, fw.c routines | core.c, fw.c             |
| TX Path              | `ath12k_mac_tx()`, TX descriptor     | mac.c, hw.c, hal.c       |
| RX Path              | `ath12k_process_rx()`, `hal_rx_parse()` | irq.c, hal.c          |
| mac80211 Ops         | `mac_op_scan()`, `sta_add()`, `assoc()` | mac.c, wmi.c          |
| Recovery             | `ath12k_core_recover()`, irq.c       | core.c, irq.c            |
| Removal              | `ath12k_pci_remove()`, `ath12k_core_stop()` | pci.c, core.c      |

---
