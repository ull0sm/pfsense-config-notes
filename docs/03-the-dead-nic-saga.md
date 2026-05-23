# 03. The Dead NIC Saga

> A router needs two network interfaces: WAN (Internet) and LAN (Local). My Proxmox PC only had one built-in ethernet port. This documents the hardware struggle to add a second interface for pfSense.

---

## What I Wanted To Do

To make pfSense a real router, I needed to assign it two virtual bridges in Proxmox:
- `vmbr0` (WAN) mapped to the built-in ethernet port.
- `vmbr1` (LAN) mapped to a second physical port.

I found an old USB-to-Ethernet adapter lying around and plugged it into the Proxmox host. 

## What I Tried: The Investigation

I connected the USB adapter to my Proxmox PC and plugged a cable from it into my TP-Link access point. However, the TP-Link router showed the port as "unplugged," and no traffic was flowing.

I jumped into the Proxmox shell to debug:

```bash
# 1. Check if Linux detects the USB device
root@pve:~# lsusb
Bus 004 Device 002: ID 0bda:8153 Realtek Semiconductor Corp. RTL8153 Gigabit Ethernet Adapter

# 2. Check the interface state
root@pve:~# ip a
3: enx00e04c314578: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN

# 3. Try to manually bring the interface UP
root@pve:~# ip link set enx00e04c314578 up
```

## What Failed

Even after running `ip link set up`, nothing changed. 
- The `ip a` command still showed no link.
- `dmesg | grep r8152` showed the driver loaded, but no link was detected.
- The TP-Link router still reported:
  ```
  Ethernet Status:
  LAN1: unplugged
  LAN2: unplugged
  LAN3: unplugged
  ```

**The Physical Check:** I finally looked closely at the adapter itself. There were absolutely no LED indicator lights turning on when the ethernet cable was plugged in, regardless of which cable or port I used.

## The Fix

> [!SUCCESS]
> **Root Cause:** The USB-to-Ethernet adapter was physically defective. It was dead hardware.

I ordered a **new USB-to-Ethernet adapter** via a quick commerce app. When the new adapter arrived:
- The LEDs lit up immediately upon plugging in the cable.
- Proxmox showed `state UP`.
- The TP-Link interface immediately changed to `LAN1: connected`.
- pfSense LAN started routing traffic.

> [!TIP]
> **Lesson Learned:** Before spending hours debugging software, virtual bridges, and kernel drivers, **check if the hardware actually works.** No LEDs usually means dead hardware.
