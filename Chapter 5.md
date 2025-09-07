# Chapter 5: High-Level RX Flow

## Step-by-Step Trace: RX Path (ath12k)

### Step 0: Packet received by hardware (PHY)
- The radio receives a frame over the air.
- Hardware processes the frame (decryption, header parsing, etc.).
- It writes the data + metadata into host memory via DMA, using buffers the driver previously posted.

### Step 1: RX buffer posted by driver
**Function:** `ath12k_dp_rx_bufs_replenish()`
- Called during init and dynamically.
- Allocates an `skb`, maps it for DMA, and posts it to the RXDMA ring (via a descriptor).
- Provides hardware a place to DMA received packets.

### Step 2: DMA complete, packet ready
- Hardware fills in:
  - Data buffer (via DMA)
  - Completion descriptor in REO ring (`REO_DST_RING`)

### Step 3: NAPI poll occurs
**Function:** `ath12k_ahb_ext_grp_napi_poll()` → `ath12k_dp_service_srng()`
- Driver is polled via NAPI (interrupt was raised earlier).
- Scans RX rings (SRNGs) such as `dp->reo_dst_ring[]`.

### Step 4: REO descriptor processed
**Function:** `ath12k_dp_rx_process_received_packets()`
- Pulls the next descriptor entry from the REO destination ring.
- Extracts physical address of data buffer (filled by hardware).
- Calls `dma_unmap_single()` to prepare the buffer for CPU use.
- Wraps the data into an `skb`.

### Step 5: RX header and metadata parsing
**Functions involved:** `ath12k_dp_rx_process_msdu()`
- Validate the packet using RX descriptors (e.g., `msdu_done` bit).
- Calculate payload length, padding, etc.
- Handle buffer coalescing (if the frame is split across buffers).
- Extract metadata (RSSI, MCS, etc.).
- Populate `ieee80211_rx_status`, which mac80211 uses.

### Step 6: Frame handed to mac80211
**Function:** `ieee80211_rx_napi(ar->hw, skb, &ar->dp.napi)`
- The packet is finally handed off to mac80211.  
- From here, it goes through:
  - Filtering (e.g., monitor, STA, AP modes)
  - Defragmentation
  - Authentication checks
  - Then enters the upper network stack
---

## RX Path Summary Diagram

### RX Packet Flow (ath12k)
```text
RX Packet Flow (ath12k)
├─ Over-the-Air Packet
├─ PHY / MAC Layer
├─ RXDMA Ring
│  └─ DMA to host buffer
├─ REO Ring
│  └─ Completion descriptor generated
├─ NAPI Poll
│  └─ Triggered via ath12k_hif_poll()
├─ ath12k_dp_service_srng()
│  └─ Processes SRNG entries
├─ ath12k_dp_rx_process_recv_buf()
│  └─ Parses descriptor, prepares SKB
├─ SKB Creation
│  └─ Metadata parsed (RSSI, rate, etc.)
├─ ieee80211_rx_napi()
│  └─ Passes SKB to mac80211
└─ Network Stack
   └─ Packet enters IP/TCP layers and userspace
```

### Interrupt Handling and NAPI Poll Flow
```ascii
Device generates interrupt
├─ ath12k_hif_ahb_interrupt_handler()
│  ├─ Identifies the IRQ source
│  └─ Calls napi_schedule(&napi->napi)
├─ SoftIRQ context (net_rx_action) runs
│  └─ Invokes ath12k_ahb_ext_grp_napi_poll()
└─ ath12k_ahb_ext_grp_napi_poll()
   └─ Calls ath12k_dp_service_srng()
      ├─ Processes RX descriptors
      └─ Handles TX completions, if any
```

### REO Descriptor Processing Flow (high-level)
```ascii
├─ Packet received over-the-air by the PHY/MAC
├─ RXDMA engine performs DMA of packet into host buffer
├─ REO hardware generates a REO completion descriptor
├─ Descriptor placed in REO ring (hardware-managed queue)
├─ NAPI poll function scheduled (e.g., ath12k_ahb_ext_grp_napi_poll)
├─ Driver processes REO descriptors via:
│ └─ ath12k_dp_service_srng()
│     └─ ath12k_dp_rx_process()
|         └─ ath12k_dp_rx_process_received_packets()  
│
│
├─ For each REO descriptor:
│   ├─ Metadata is extracted (e.g., status, length, peer ID, TID)
│   ├─ Corresponding skb is retrieved or created
│   ├─ Status and errors are validated
│   ├─ Packet flags (e.g., decrypted, QoS, MIC fail) are parsed
│   └─ skb is populated with metadata
├─ skb is passed to mac80211 via ieee80211_rx_napi()
└─ Packet continues up the Linux networking stack (IP, TCP, userspace)
```

### RX header and metadata parsing
```ascii
ath12k_dp_rx_process_received_packets()
└── for each msdu in list:
    └── ath12k_dp_rx_process_msdu()
    │    └── Parses, validates, and populates rx_info
    └── ath12k_dp_rx_deliver_msdu()
         └── ieee80211_rx_napi()
```