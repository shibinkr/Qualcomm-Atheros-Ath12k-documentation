# Chapter 2: PCI & AHB Probe Functions

## 1. Overview

The **probe functions** — `ath12k_pci_probe()` and `ath12k_ahb_probe()` —  
are the **real entry points** where hardware initialization begins.  

When the kernel detects a supported **ath12k device** on a given bus (PCI or AHB/SoC),  
it calls the corresponding probe function. This probe routine is responsible for:

- **Enabling the hardware** (turning on the device and bus access).  
- **Allocating system resources** such as memory regions and IRQs.  
- **Configuring DMA** so the device and host can exchange buffers.  
- **Creating the core driver context** (`ath12k_core_create()`).  
- **Loading and starting firmware** (`ath12k_core_start()`).  
- **Saving driver state** for future operations and cleanup.  

If any of these steps fail, the probe must gracefully **free resources** and return an error.

---

## 2. Detailed Flow of `ath12k_pci_probe()`

When a PCI-based ath12k device is detected, the kernel invokes `ath12k_pci_probe()`.  
This function performs the following sequence:

1. **Enable the PCI device**  
   - Call PCI core helpers (`pci_enable_device()`) to turn on the device.  
   - Configure DMA masks so the device can access host memory correctly.  

2. **Allocate system resources**  
   - Request **IRQ vectors** (MSI/MSI-X or legacy).  
   - Allocate device-private memory structures for state tracking.  

3. **Map PCI BAR regions**  
   - The Base Address Registers (BARs) expose device MMIO regions.  
   - These are mapped into kernel virtual memory using `pci_iomap()`.  
   - Provides access to device registers for later control.  

4. **Initialize driver core**  
   - Call `ath12k_core_create()` to allocate and initialize the main driver object (`struct ath12k`),  
     set up DMA rings, workqueues, WMI/HTT skeletons, and mac80211 structures.  

5. **Start firmware**  
   - Invoke `ath12k_core_start()`.  
   - Downloads firmware, performs the WMI/HTT handshake, enables data paths, and registers with mac80211.  

6. **Save driver state**  
   - Store a reference to the initialized `ath12k` context in the PCI device data (`pci_set_drvdata()`),  
     allowing future remove/shutdown routines to find and tear down the driver.  

7. **Handle failures**  
   - If any step fails (e.g., DMA mapping error, firmware load failure), the function must:  
     - Free BAR mappings.  
     - Release IRQs.  
     - Destroy workqueues and allocated memory.  
   - Return an error code to the kernel PCI subsystem.  

---

## 3. Detailed Flow of `ath12k_ahb_probe()`

For SoC-based ath12k devices connected via the **AHB bus**, the kernel calls `ath12k_ahb_probe()`.  
This flow is similar in spirit to PCI but tailored to on-chip integration.

1. **Setup DMA mask**  
   - Ensure the platform device supports coherent DMA addressing for buffer sharing.  

2. **Allocate driver structures**  
   - Allocate `struct ath12k` and supporting state memory.  

3. **Detect hardware revision**  
   - Read SoC hardware IDs.  
   - Assign function pointers (`ops`) specific to the hardware generation or revision.  

4. **Map device resources**  
   - Map physical memory ranges into kernel virtual space (from `platform_get_resource()`).  
   - Initialize **clocks** and **resets** needed to power the WLAN block.  

5. **Setup Copy Engine (CE) and hardware rings**  
   - Initialize CE pipes, which provide DMA-based communication between host and firmware.  
   - Allocate RX/TX rings for packet data.  

6. **Initialize QMI and rproc firmware interface**  
   - QMI (Qualcomm Messaging Interface) is used for firmware control and communication.  
   - rproc (remote processor) APIs may be used to boot WLAN firmware running on a separate CPU core.  

7. **Configure interrupts**  
   - Request IRQs from the platform and register ISR handlers.  

8. **Initialize driver core & load firmware**  
   - Call `ath12k_core_create()` to prepare driver state.  
   - Call `ath12k_core_start()` to download firmware, handshake, and register with mac80211.  

9. **Save driver data**  
   - Store the driver context pointer in the platform device’s private data (`platform_set_drvdata()`).  

10. **Error handling**  
    - If any step fails (e.g., clock init failure, QMI timeout, firmware load error),  
      release memory, unmap I/O regions, disable clocks, and return an error.  

---

## 4. Key Difference: PCI vs AHB Probe

- **PCI path:**  
  - Focuses on enabling/disabling a discrete PCIe device.  
  - Uses **BAR mappings** and PCI subsystem helpers.  
  - Firmware is loaded via host-controlled PCI bus transfers.  

- **AHB path:**  
  - Integrated on-chip, often tied to SoC clock/reset frameworks.  
  - Relies on **QMI/rproc** for firmware management.  
  - Needs clock/reset handling in addition to DMA/interrupt setup.  

In both cases, the **end goal** is the same:  
a successfully created `ath12k` core context, firmware running, and device registered with **mac80211**.

