# Dell Precision T7810 Home Server Hardware Guide

## System Specifications

### Base System: Dell Precision T7810
- **CPU**: 2x Xeon E5-2620 v3 (6 cores/12 threads each, total 12 cores/24 threads)
- **RAM**: 32GB DDR4 ECC (upgradable to 512GB across 8 DIMM slots)
- **Storage**: 500GB HDD (upgradable, supports multiple drives)
- **Chipset**: Intel C612
- **PCIe Slots**: 
  - 3x PCIe 3.0 x16 slots
  - 2x PCIe 3.0 x4 slots
- **Power Supply**: 685W or 1125W (depending on configuration)
- **Form Factor**: Tower workstation with excellent cooling

## Compatible Hardware Components

### 1. RM520 Cellular Modem

#### Recommended: Quectel RM520N-GL
**Specifications:**
- 5G NR Sub-6GHz modem
- M.2 form factor (Key B, 3052)
- Supports SA/NSA modes
- Dual SIM support
- Download: Up to 4.0 Gbps
- Upload: Up to 900 Mbps

**Installation Requirements:**
- **M.2 to PCIe Adapter**: Needed since T7810 doesn't have native M.2 slots
  - Recommended: NGFF M.2 Key B to PCIe x4 adapter card
  - Models: IOCREST IO-PCE1061-2B or similar
  - Uses one PCIe x4 slot

**Additional Components:**
- **Antennas**: 4x external 5G antennas (MIMO)
  - Recommended: Waveform 4x4 MIMO antenna kit
  - SMA or IPEX connectors (check modem variant)
- **SIM Card**: Compatible with your cellular provider
- **USB Adapter** (alternative): Quectel RM520N-GL in USB enclosure
  - Easier installation but potentially lower performance
  - Models: Quectel EVB kit or third-party USB enclosures

### 2. WiFi 7 Network Card

#### Recommended Options:

**Option A: Intel BE200 (Best for Linux/Stability)**
- **Chipset**: Intel BE200
- **Standard**: WiFi 7 (802.11be)
- **Speed**: Up to 5.8 Gbps
- **Form Factor**: M.2 2230 (requires M.2 to PCIe adapter)
- **Linux Support**: Excellent (kernel 6.2+)
- **Features**:
  - 2.4GHz, 5GHz, 6GHz bands
  - MU-MIMO, OFDMA
  - Bluetooth 5.4

**Installation**:
- M.2 2230 to PCIe x1 adapter
- Recommended adapter: EDUP M.2 WiFi 6E to PCIe adapter
- External antennas (2-4x RP-SMA)

**Option B: MediaTek MT7925**
- Similar specs to Intel BE200
- Good Linux support (kernel 6.5+)
- Often found in Fenvi cards

**Option C: Qualcomm FastConnect 7800**
- Highest performance WiFi 7 chipset
- Limited Linux driver support (improving)
- Better for Windows-based setup

**Recommended Complete Cards:**
1. **Fenvi FE1BE** (Intel BE200 based)
   - PCIe x1 interface (direct installation)
   - Includes external antennas
   - Bluetooth 5.4
   - Price: ~$50-70

2. **TP-Link Archer TXE75E**
   - PCIe x1 interface
   - WiFi 7 with external antennas
   - Good Windows/Linux support
   - Price: ~$80-100

3. **ASUS PCE-BE92BT**
   - High-end option
   - PCIe x1
   - 6 external antennas
   - Bluetooth 5.4
   - Price: ~$120-150

### 3. Additional Storage (Recommended)

Since you'll be running LANCache, additional storage is highly recommended:

**SSD for OS and Services:**
- **500GB-1TB NVMe SSD** via M.2 to PCIe adapter
  - Recommended: Samsung 970 EVO Plus 1TB
  - Kingston KC3000 1TB
  - Uses one PCIe x4 slot

**HDD/SSD for LANCache:**
- **2-4TB SSD** for optimal performance
  - Recommended: Crucial MX500 4TB (SATA)
  - Samsung 870 QVO 4TB (SATA)
- **Alternative**: 4-8TB HDD for budget option
  - WD Red Plus 4TB
  - Seagate IronWolf 4TB
- T7810 supports multiple 3.5" and 2.5" drive bays

### 4. Network Interface Card (Optional but Recommended)

For better network segmentation and performance:

**Intel I350-T4**
- 4x Gigabit Ethernet ports
- PCIe x4
- Excellent Linux support
- Allows separate VLANs for management, LAN, WAN
- Price: ~$100-150 (used/refurbished market)

**Alternative: Intel X550-T2**
- 2x 10 Gigabit Ethernet ports
- For high-performance setups
- Price: ~$150-200 (used market)

## PCIe Slot Allocation Plan

Based on available slots:

1. **PCIe Slot 1 (x16)**: WiFi 7 Card (Fenvi FE1BE or similar)
2. **PCIe Slot 2 (x4)**: RM520 5G Modem (via M.2 adapter)
3. **PCIe Slot 3 (x4)**: NVMe SSD (via M.2 adapter) for OS
4. **PCIe Slot 4 (x16)**: Additional NIC (optional)
5. **PCIe Slot 5 (x16)**: Available for future expansion

## Power Requirements

Total estimated power draw:
- System base: ~150W idle, ~300W load
- RM520 modem: ~10W
- WiFi 7 card: ~15W
- Additional drives: ~10W each
- **Total**: ~200W idle, ~350W under load

The 685W PSU is sufficient for this configuration.

## Cooling Considerations

- T7810 has excellent airflow with large front fans
- Ensure proper cable management for airflow
- Monitor temperatures, especially for:
  - RM520 modem (may need heatsink)
  - WiFi 7 card under heavy load
  - NVMe SSD (use adapter with heatsink)

## Complete Shopping List

### Essential Components:

1. **5G Modem Setup**: $200-300
   - Quectel RM520N-GL module: ~$120-150
   - M.2 Key B to PCIe adapter: ~$20-30
   - 4x 5G MIMO antennas: ~$60-100

2. **WiFi 7 Card**: $50-150
   - Fenvi FE1BE or TP-Link TXE75E: ~$50-100

3. **Storage Upgrade**: $150-400
   - 1TB NVMe SSD + adapter: ~$80-120
   - 4TB SATA SSD for LANCache: ~$200-300

4. **Optional NIC**: $100-150
   - Intel I350-T4: ~$100-150

### Supporting Components:

5. **Cables and Adapters**: $30-50
   - Antenna extension cables (if needed): ~$10-20
   - SATA cables: ~$10
   - Ethernet cables (Cat6/Cat6a): ~$10-20

6. **Cooling Enhancements** (optional): $20-50
   - M.2 heatsinks: ~$10-15
   - Case fans (if needed): ~$10-20
   - Thermal pads: ~$10

**Total Estimated Cost**: $600-1,100 depending on options chosen

## Compatibility Notes

### BIOS Considerations:
- Update to latest BIOS before installation (check Dell support site)
- Enable UEFI boot mode
- Disable Secure Boot if using Linux
- Enable virtualization (VT-x, VT-d) for containers/VMs

### Operating System Compatibility:
- **Linux (Recommended)**:
  - Ubuntu Server 24.04 LTS
  - Debian 12
  - Proxmox VE 8.x (for virtualization)
- **Windows Server 2022** (alternative)

### Known Issues:
- Some M.2 adapters may require specific PCIe slot assignment
- RM520 may need manual APN configuration
- WiFi 7 features require kernel 6.2+ on Linux
- Check cellular band compatibility with your provider

## Firmware and Drivers

### RM520 Modem:
- Update to latest firmware from Quectel
- Use ModemManager or QMI tools on Linux
- Install Quectel drivers on Windows

### WiFi 7 Card:
- Linux: Use kernel 6.2+ for native support
- Windows: Install vendor drivers
- Update firmware via vendor tools

### Network Card:
- Linux: Native drivers in kernel
- Windows: Intel drivers from official site

## Alternative Configurations

### Budget Option (~$300):
- Use USB version of RM520
- WiFi 6E card instead of WiFi 7 (Intel AX210)
- Keep existing HDD, add single SSD
- No additional NIC

### High-Performance Option (~$1,500):
- Add 10GbE NIC
- Multiple NVMe SSDs in RAID
- 8TB+ SSD for LANCache
- Enterprise WiFi 7 AP instead of PCIe card
- Additional RAM upgrade to 64GB+

## Next Steps

1. Verify current BIOS version and update if needed
2. Check cellular provider band compatibility
3. Order components based on budget
4. Prepare installation tools (screwdrivers, thermal paste, etc.)
5. Review the SOFTWARE_SETUP.md guide for configuration instructions
