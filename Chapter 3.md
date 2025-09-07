# Chapter 3 — `ath12k_core_create()` & `ath12k_core_start()`

This chapter explains the **two most important functions** in `core.c`:  
`ath12k_core_create()` (driver core setup) and `ath12k_core_start()` (firmware bring-up).

---

## 1. `ath12k_core_create()` — Core Initialization

This function prepares all the **software-side structures** needed before firmware can run.  
It does not yet power up or register the device — it just sets the stage.

- **Allocate driver data**  
  Creates `struct ath12k`, holding all per-device state, bus references, and synchronization objects.  

- **Set up workqueues**  
  Allocates kernel workqueues for tasks like firmware events, recovery, and async jobs.  

- **Initialize RX/TX rings**  
  Allocates DMA-coherent memory for transmit and receive descriptors.  
  RX buffers are pre-filled and mapped so firmware can DMA packets directly.  

- **Configure control protocols**  
  Prepares WMI (control/config messaging) and HTT (data transport setup).  
  Registers event handlers but does not yet start communication.  

- **mac80211 integration**  
  Creates `ieee80211_hw` and sets up basic callbacks and wireless capabilities (bands, rates, interface modes).  

- **Firmware bookkeeping**  
  Prepares firmware filenames, board data references, and supporting structures for later download.  

At the end of `ath12k_core_create()`, the driver has a fully allocated **but inactive** context.  
If this step fails (e.g., memory/DMA allocation issues), probe aborts early.

---

## 2. `ath12k_core_start()` — Firmware Bring-up

Once core structures exist, this function actually **boots the hardware**.

- **Firmware load**  
  Requests the right firmware binary via the kernel firmware API (or QMI/rproc on SoC).  
  Transfers it to device memory and waits for a "firmware ready" signal.  

- **WMI handshake**  
  Sends initial setup commands (capabilities, feature negotiation).  
  Registers handlers for firmware events and management responses.  

- **HTT setup**  
  Negotiates data path parameters (queue sizes, buffer formats).  
  Enables RX/TX rings and configures DMA windows.  

- **mac80211 registration**  
  Completes population of `ieee80211_hw` capabilities (using firmware-reported values)  
  and registers with mac80211. At this point, the device becomes visible to userspace (`iw`, `NetworkManager`).  

- **Start data paths**  
  Enables interrupts, activates TX/RX paths, and starts background firmware tasks (calibration, monitoring).  

When `ath12k_core_start()` succeeds, the device is fully **operational**.  
If it fails (e.g., firmware missing, handshake timeout), cleanup rolls back to a safe state.

---

## 3. Lifecycle Notes

- `ath12k_core_create()` = **software readiness** (memory, rings, queues, skeleton objects).  
- `ath12k_core_start()` = **hardware/firmware activation** (firmware load, handshakes, registration).  
- Both functions have strict error handling paths to unwind allocations and avoid leaks.  

---
