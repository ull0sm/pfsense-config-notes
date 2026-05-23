# 🔩 Hardware & Specs

> Every piece of hardware used in this project.

---

## The Server — Proxmox Host

| Component | Details |
|-----------|---------|
| Type | Spare PC repurposed |
| Role | Hypervisor running all VMs |
| OS | Proxmox VE (Debian-based) |
| Built-in NIC | Realtek (MAC: `00:e0:4c:58:fa:bf`) |
| NIC Interface | `nic0` → bridged to `vmbr0` |

---

## Network Interfaces

### NIC 1: Built-in Ethernet (WAN)

| Detail | Value |
|--------|-------|
| Chipset | Realtek |
| Interface Name | `nic0` |
| Alt Name | `enx00e04c58fabf` |
| MAC Address | `00:e0:4c:58:fa:bf` |
| Bridge | `vmbr0` |
| Role | **WAN** — connects to Airtel router |
| Speed | Gigabit |
| Status | UP, working |

### NIC 2: USB Ethernet Adapter (LAN)

| Detail | Value |
|--------|-------|
| Chipset | **Realtek RTL8153** |
| Interface Name | `enx00e04c314578` |
| MAC Address | `00:e0:4c:31:45:78` |
| Connection | USB 3.0 |
| Bridge | `vmbr1` |
| Role | **LAN** — connects to TP-Link AP |
| Speed | Gigabit |
| Linux Driver | `r8152` kernel module |

> **Note:** The first USB adapter was defective (non-functional, no link LED). Had to order a replacement via quick commerce. The replacement RTL8153 works perfectly.

#### USB Detection

```bash
root@pve:~# lsusb
Bus 004 Device 002: ID 0bda:8153 Realtek Semiconductor Corp. RTL8153 Gigabit Ethernet Adapter
```

#### Bringing Up the Interface

```bash
# Check interface status
ip a
# Shows: enx00e04c314578: state DOWN

# Bring it up
ip link set enx00e04c314578 up

# Verify link
ethtool enx00e04c314578
# Link detected: yes
```

### NIC That Was NOT Used

| Detail | Value |
|--------|-------|
| Device | TP-Link AC1300 Archer T3U Plus |
| Type | USB 3.0 Wi-Fi Dongle |
| Interface | `wlx74feced44371` |
| Status | **NOT USED** — state DOWN |

**Why not used:**
- Realtek WiFi chipset → poor FreeBSD/pfSense support
- Unstable AP mode
- Poor throughput
- Random disconnects
- pfSense is designed for Intel/Ethernet NICs, not consumer USB WiFi dongles

---

## Airtel ISP Router/ONT

| Detail | Value |
|--------|-------|
| Provider | Airtel (India) |
| Type | ISP-provided combo (ONT + Router + WiFi) |
| Connection | FTTH (Fiber to the Home) |
| WAN Type | PPPoE |
| LAN IP | `192.168.1.1` |
| Public IP | `122.172.83.46` (dynamic) |
| Current Role | Modem + upstream NAT + PPPoE |
| Previous Role | Main router/firewall/DHCP/WiFi (everything) |

### Limitations That Caused This Project

- ❌ Port 443 reserved for remote management
- ❌ Locked firmware
- ❌ No VLAN support
- ❌ Poor DNS control
- ❌ No bridge mode (or hard to enable)

---

## TP-Link Router (Access Point)

| Detail | Value |
|--------|-------|
| Brand | TP-Link |
| Ports | 1 WAN + 4 LAN |
| Current Mode | Access Point (manual config) |
| IP | `10.27.27.2` |
| DHCP | **DISABLED** |
| Role | WiFi only — no routing, no DHCP |
| Connected via | LAN port → USB Ethernet → Proxmox vmbr1 |

### Configuration

- NOT using built-in "AP Mode" (it was unreliable)
- Manually configured: DHCP off, static IP `10.27.27.2`
- Cable plugged into **LAN port** (NOT WAN/Internet port)

---

## pfSense VM Specifications

| Setting | Value |
|---------|-------|
| VM ID | Assigned in Proxmox |
| BIOS | OVMF (UEFI) |
| Machine | q35 |
| CPU | 2 cores |
| RAM | 4 GB |
| Disk | 20 GB |
| NIC 1 (WAN) | VirtIO → vmbr0 |
| NIC 2 (LAN) | VirtIO → vmbr1 |
| OS | pfSense CE (Community Edition) |

---

## Recommended Upgrades (Future)

| Priority | Upgrade | Why |
|----------|---------|-----|
| #1 | Intel PCIe NIC (i350/i210/i340) | Best pfSense compatibility, bypass USB issues |
| #2 | Dedicated WiFi Access Point | Better coverage, proper AP hardware |
| #3 | Managed Switch | VLAN support for network segmentation |

### NIC Compatibility Guide

| Chipset | pfSense Support | Notes |
|---------|----------------|-------|
| Intel i210/i225 | ⭐ Excellent | Best choice |
| Intel i350 | ⭐ Excellent | Enterprise-grade |
| Realtek RTL8153 USB | ✅ Acceptable | Current setup, works fine |
| ASIX USB | ⚠️ Meh | Inconsistent |
| Realtek WiFi (USB) | ❌ Bad | Don't use with pfSense |
