# Chapter 4: TX Path in ath12k

The **TX path** is the sequence by which outgoing network packets travel from the Linux networking stack to the hardware for over-the-air transmission. It involves **mac80211**, the **ath12k driver**, transport protocols (**HTT**), DMA engines (**SRNG rings / CE**), and the **firmware** running on the WLAN device.

---

## 1. Key Components

- **mac80211** — Linux wireless stack; hands packets down to the driver.  
- **ath12k driver** — Translates packets into hardware-specific formats, sets up descriptors.  
- **HTT (Host Target Transport)** — Host ↔ firmware protocol for data packet exchange.  
- **SRNG rings / Copy Engine (CE)** — DMA engines that move packet buffers between host and device.  
- **Firmware** — Performs rate control, aggregation, encryption, and final transmission over radio.  

---

## 2. Detailed TX Path Flow

1. **Packet Reception (mac80211 → ath12k)**  
   - Entry point: `ath12k_mac_op_tx()` (called by mac80211).  
   - Input: an `sk_buff` (socket buffer).  

2. **Queueing & Classification**  
   - Classify packet by traffic priority.  
   - Enqueue into internal software queues mapped to hardware TX rings.  

3. **HTT TX Descriptor Preparation**  
   - Build TX descriptors containing metadata (length, flags, rate info).  
   - Guides firmware on how to handle transmission.  

4. **DMA Setup (SRNG/CE)**  
   - Map packet buffers into DMA space with `dma_map_single()`.  
   - Push descriptors into **TCL SRNG rings** (Transmit Command List).  
   - CE is used mostly for control; actual TX path uses SRNG rings for speed.  

5. **Notify Firmware**  
   - Driver doorbells hardware by updating the SRNG head pointer.  
   - Firmware is alerted that new TX descriptors are ready.  

6. **Firmware Processing**  
   - Firmware dequeues packets via DMA.  
   - Handles encryption, retries, aggregation, and rate control.  
   - Transmits frames over the radio.  

7. **TX Completion**  
   - Firmware sends completion events (via HTT/WMI).  
   - Driver frees SKBs, updates statistics, and informs mac80211.  

---

## 3. Code Flow Overview

```text
mac80211
 └─> ath12k_mac_op_tx()
     ├─ drop monitor frames if needed
     ├─ handle MLO (multi-link) or single-link
     ├─ encryption / encapsulation handling
     ├─ direct TX → ath12k_dp_tx()
     │    └─ enqueue skb into TCL SRNG rings
     └─ multicast TX → replicate skb per link
          └─ per-link ath12k_dp_tx()
```
---
## 4. `ath12k_dp_tx()` — Core TX Function

This function prepares and enqueues SKBs for hardware transmission.  
Its main responsibilities are:

- **Allocate a TX descriptor (`tx_desc`)**  
- **Fill HAL TX info (`ti`)** with DMA addresses and metadata  
- **Map the buffer for DMA** using `dma_map_single()`  
- **Build MSDU/ext descriptors** if required  
- **Push descriptors** into the **TCL SRNG ring**  
- **Update the head pointer** and **doorbell hardware** to start transmission  


```text
ath12k_dp_tx()
 ├─ allocate tx_desc
 ├─ setup HAL TX info
 ├─ dma_map_single() → ti.paddr
 ├─ prepare MSDU/ext desc
 ├─ push to SRNG (TCL) ring
 │    ├─ ath12k_hal_srng_src_get_next_entry()
 │    ├─ ath12k_hal_tx_cmd_desc_setup()
 │    └─ ath12k_hal_srng_access_end()  → doorbell h/w
 ▼
 Target Firmware
   ├─ DMA fetch (ti.paddr)
   └─ transmit over air
```
---
## 5. Ring Mechanics in TX Path

TX communication between host and firmware is managed by **SRNG rings** (circular buffers).

---

### Key Pointers
- **Head Pointer (hp):** where the host writes the next descriptor.  
- **Tail Pointer (tp):** where hardware/firmware has consumed up to.  
- Both are stored in **MMIO/shared memory**, updated by host and device.  

---

## Ring Lifecycle

### Step 1: Ring Allocation & Initialization
In `ath12k_hal_srng_setup()`:
```c
srng->u.src_ring.hp = 0;
srng->u.src_ring.tp = 0;
srng->u.src_ring.hp_addr = base_vaddr + HP_OFFSET;
srng->u.src_ring.tp_addr = base_vaddr + TP_OFFSET;
```
In `ath12k_hal_srng_setup()`:
- Head pointer (`hp`) and tail pointer (`tp`) are initialized to 0.  
- `hp_addr` and `tp_addr` are virtual addresses mapped to **MMIO or shared memory**.  
- The **base address** is determined when the ring is created by the firmware or host (via `hal_ring` structures).  

---

### Step 2: Descriptor Enqueue
In `ath12k_dp_tx()`:
- A new TX descriptor (`hal_tcl_data_cmd`) is filled.  
- Pointer to the descriptor is stored in the **SRNG ring memory**.  
- The head pointer (`hp`) is incremented, wrapping around when it reaches the end of the circular buffer.  

```code
srng->u.src_ring.hp = (srng->u.src_ring.hp + 1) & ring_mask;
```
---

### Step 3: Ring Doorbell
In `ath12k_hal_srng_access_end()`:
```code
ath12k_hif_write32(..., srng->u.src_ring.hp);
```
- Host writes the updated head pointer to the hardware register.  
- This **notifies the hardware** that new descriptors are ready for processing.  

---

### Step 4: Ring Consumption (Firmware Side)
- Firmware reads descriptors from the ring via DMA.  
- Updates the **tail pointer (`tp`)** as descriptors are consumed.  
- Host software can optionally read back `tp` from `tp_addr` to track hardware progress.  
- This mechanism allows host polling and synchronization with firmware consumption.
```code
srng->u.src_ring.last_tp = *(volatile u32 *)srng->u.src_ring.tp_addr;

```
---
## 6. Key Takeaways

- **SRNG rings** (e.g., TCL) are the main TX mechanism; **CE** is used mainly for control.  
- `dma_map_single()` ensures SKB data is visible to the device.  
- Descriptor rings act as **queues for host ↔ firmware communication**.  
- **TX completion events** allow resource cleanup and statistics updates.  
