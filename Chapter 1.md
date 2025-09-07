# Chapter 1: Module Load & Driver Registration — Basic Details

## 1. Driver Registration in `core.c`

- The **core of ath12k** is initialized via `ath12k_<bus>_register()` in `core.c`.  
- This function registers the **common wireless operations and interfaces** used by both **PCI** and **AHB (Advanced High-Performance Bus)** variants.  
- It sets up:
  - Core operations callbacks.  
  - WMI and HTT communication interfaces.  
  - Generic driver features.  

⚠️ Note: `ath12k_register()` itself is **not** the entry point for the kernel module load — it is called by the **bus-specific probe functions** (`pci.c` or `ahb.c`).  

---

## 2. Platform-Specific Driver Registration

- The **real module entry points** are in `pci.c` and `ahb.c`:  
  - `pci.c` → calls `pci_register_driver(&ath12k_pci_driver)` inside `ath12k_pci_init()`.  
  - `ahb.c` → calls `platform_driver_register(&ath12k_ahb_driver)` inside `ath12k_ahb_init()`.  

- Both register **callbacks** to probe and remove the device.

---

## 3. Probe Function (`ath12k_pci_probe()` or `ath12k_ahb_probe()`)

- When the device is detected, these **probe functions** are invoked.  
- Inside the probe:
  - Acquire resources (memory, IRQ).  
  - Setup DMA.  
  - Call `ath12k_core_create()` — **the key function in `core.c`** that allocates and initializes the main `ath12k` structure and sets up core wireless components.  
  - After creation, call `ath12k_core_start()` — boots the firmware, initializes hardware, and registers the wireless device with **mac80211**.  

---

## 4. `ath12k_core_create()` (Core Initialization in `core.c`)

- Allocates driver data (`struct ath12k`).  
- Sets up **workqueues** for async work.  
- Initializes **RX/TX rings** (hardware queues).  
- Initializes **WMI**, **HTT**, and other control protocols.  
- Sets up **mac80211 wireless hardware capabilities**.  
- Prepares **firmware structures** for loading.  

---

## 5. `ath12k_core_start()` (Firmware Start)

- Downloads the firmware.  
- Sends **initialization commands** via WMI.  
- Registers the wireless device (**wiphy**) with the **mac80211 stack**.  
- Starts **data paths (TX/RX)**.  
- Once complete, the device is **operational**.  

---

## 6. Module Removal

- On driver unload or device removal:  
  - Bus-specific remove functions call `ath12k_core_stop()` and `ath12k_core_destroy()`.  
  - These functions:
    - Halt data paths.  
    - Unregister the device.  
    - Free IRQs.  
    - Release memory.  

---

## Summary Flow (with focus on `core.c`)

```text
Module load
   ↓
Bus-specific init (pci.c or ahb.c)
   ↓
Driver registration (pci_register_driver / platform_driver_register)
   ↓
Device detected → probe function
   ↓
ath12k_core_create()
   (allocates and initializes core)
   ↓
ath12k_core_start()
   (downloads firmware and registers mac80211 device)
   ↓
Device operational
```