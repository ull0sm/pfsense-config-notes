# Homelab Network Documentation

> **A raw, technical documentation of my attempt to learn networking by building a pfSense router in my homelab.**
>
> This project was built as a stepping stone to understanding network fundamentals, as part of my journey into SRE and Platform Engineering. No corporate hype, just the exact debugging steps, commands, and failures I hit along the way.

---

## 📋 Overview

I set out to host some web apps on a spare PC running Proxmox. In the process of trying to open ports, I hit a hard limitation with my ISP-provided router's firmware.

Instead of working around it, I decided to virtualize my network routing using **pfSense**. This repository documents the exact struggles I faced and the commands I used to fix them.

---

## 📚 Documentation (The War Stories)

I've documented the entire setup process focusing on **What I wanted → What I tried → What failed → How I fixed it**.

| # | Document | What It Covers |
|:---:|:---|:---|
| **01** | [The Goal & Architecture](docs/01-the-goal.md) | The initial goal, physical cables, logical Proxmox bridges, and the final network diagram. |
| **02** | [The ISP Wall](docs/02-the-isp-wall.md) | Testing for CGNAT (`curl ifconfig.me`), NAT loopback failures, and why the Airtel router's "Remote MGMT" port 443 block forced me to use pfSense. |
| **03** | [The Dead NIC Saga](docs/03-the-dead-nic-saga.md) | Debugging a dead USB-to-Ethernet adapter using `lsusb`, `ip a`, and `state DOWN`, and replacing the hardware. |
| **04** | [The TP-Link AP Struggle](docs/04-tp-link-ap-struggle.md) | How the vendor's buggy "AP Mode" killed the network, and the manual workaround (disabling DHCP + LAN-to-LAN bridging). |
| **05** | [Debug Cheat Sheet](docs/05-debug-cheat-sheet.md) | A master list of all the CLI commands (`ss -tulnp`, `ethtool`, `ip link set up`) used to debug these issues. |

---

## 🛠️ Tech Stack Used

- **Hypervisor:** Proxmox VE
- **Firewall/Router:** pfSense CE (Virtualized)
- **Networking:** VirtIO Bridges (`vmbr0`, `vmbr1`)
- **Hardware:** Spare PC, USB-to-Ethernet Adapter, TP-Link Router (as AP)

---

## 🌍 Context

- **Location:** India
- **ISP:** Airtel FTTH (PPPoE connection, real Public IP confirmed)
- **Goal:** Learn how traffic routes from the internet to internal services by controlling the firewall directly.

---

*This is a living document of my homelab learning process. The struggle is the documentation.*
