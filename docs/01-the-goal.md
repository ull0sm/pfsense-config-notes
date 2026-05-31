#  01. The Goal & Architecture

> **TL;DR:** I had a spare PC, Proxmox, and a dream. My ISP had other plans. This is how I ended up virtualizing my own firewall to get around it.

---

## What I Wanted To Do

It started with a very simple goal:

> *"I have a spare PC with Proxmox, and I want to host my web apps with custom domains."*

### What I had

| Thing | Details |
|:---|:---|
|  **Proxmox VE** | A spare PC turned hypervisor |
| **Airtel FTTH** | ISP in India — thought it was a real public IP (it wasn't) |
| 🧠 **Networking knowledge** | "Plug in ethernet, internet works" |

### What I wanted

- `myapp.mydomain.com` → my self-hosted app
- Port forwarding so the world can hit my server

Simple enough, right? **Wrong.** → See [02 — The ISP Wall](02-the-isp-wall.md)

---

## 🏗 The Final Architecture

After a week of debugging and fighting my ISP router, I ended up virtualizing a **pfSense** firewall inside Proxmox. Here's the complete picture.

---

### Physical Cable Diagram

How the actual hardware is wired up:

```mermaid
flowchart TD
    NET(("Internet")) --> AIRTEL

    AIRTEL["**Airtel Router**
    ISP Router
    DHCP · NAT · Firewall · WiFi"]

    AIRTEL -->|"Airtel WiFi\n(phones, TVs, laptops)"| AWIFI(("Normal\nDevices"))
    AIRTEL -->|"LAN Cable"| NIC0

    NIC0["Built-in Ethernet
    nic0 → **vmbr0** (WAN)"]

    NIC0 --> PVE

    PVE[" **Proxmox VE Host**
    ┌──────────────────────┐
    │  vmbr0 = WAN bridge  │
    │  vmbr1 = LAN bridge  │
    └──────────────────────┘"]

    PVE -->|"USB Ethernet Adapter\nenxXXXXXXXXXXXX → vmbr1"| TPLINK

    TPLINK["**TP-Link Router**
    Access Point Mode
    DHCP: Disabled"]

    TPLINK -->|"TP-Link WiFi"| LWIFI(("Protected\nDevices"))

    style NET fill:#2d3436,stroke:#dfe6e9,color:#fff,font-weight:bold
    style AIRTEL fill:#0984e3,stroke:#74b9ff,color:#fff
    style NIC0 fill:#636e72,stroke:#b2bec3,color:#fff
    style PVE fill:#00b894,stroke:#55efc4,color:#fff,font-weight:bold
    style TPLINK fill:#e17055,stroke:#fab1a0,color:#fff
    style AWIFI fill:#74b9ff,stroke:#0984e3,color:#2d3436
    style LWIFI fill:#6c5ce7,stroke:#a29bfe,color:#fff
```

**Hardware → Bridge Mapping:**

| Physical Interface | Linux Name | Proxmox Bridge | Role |
|:---|:---|:---|:---|
| Built-in Ethernet | `nic0` | `vmbr0` | WAN — connects to Airtel |
| USB Ethernet Adapter | `enxXXXXXXXXXXXX` | `vmbr1` | LAN — connects to TP-Link AP |

---

### 🧠 Logical / IP Architecture

How the IP addresses and subnets are structured — there are **two completely separate networks**:

```mermaid
flowchart TD
    subgraph WAN [" AIRTEL NETWORK — 192.168.1.0/24 (vmbr0)"]
        direction TB
        INET(("Internet\nPublic IP")) --> ISP

        ISP["Airtel Router
        192.168.1.1
        NAT · DHCP · Firewall"]

        ISP --> PVE_HOST & UBUNTU & PF_WAN & OTHER

        PVE_HOST[" Proxmox Host
        192.168.1.240
        vmbr0"]

        UBUNTU["Ubuntu VM
        192.168.1.18
        ens18
        ⚠ NOT behind pfSense"]

        PF_WAN["pfSense WAN
        192.168.1.x
        vmbr0 NIC"]

        OTHER(("Other Airtel
        Devices"))
    end

    subgraph LAN ["pfSense PROTECTED NETWORK — 192.168.10.0/24 (vmbr1)"]
        direction TB
        PF_LAN["pfSense LAN
        192.168.10.1
        Firewall · DHCP · DNS"]

        PF_LAN -->|"vmbr1 → USB Ethernet\nenxXXXXXXXXXXXX"| TPLINK

        TPLINK["TP-Link AP
        Access Point
        DHCP: Off"]

        TPLINK --> C1 & C2 & C3

        C1(("Laptop\n192.168.10.x"))
        C2(("Phone\n192.168.10.x"))
        C3((" Other\n192.168.10.x"))
    end

    PF_WAN -.->|"pfSense bridges\nWAN ↔ LAN"| PF_LAN

    style WAN fill:#eaf4ff,stroke:#0984e3,color:#2d3436
    style LAN fill:#f0fff4,stroke:#00b894,color:#2d3436
    style ISP fill:#0984e3,stroke:#74b9ff,color:#fff
    style UBUNTU fill:#fdcb6e,stroke:#e17055,color:#2d3436,font-weight:bold
    style PF_WAN fill:#d63031,stroke:#ff7675,color:#fff
    style PF_LAN fill:#d63031,stroke:#ff7675,color:#fff
    style TPLINK fill:#e17055,stroke:#fab1a0,color:#fff
    style PVE_HOST fill:#00b894,stroke:#55efc4,color:#fff
    style INET fill:#2d3436,stroke:#dfe6e9,color:#fff
```

---

### IP Address Summary

| Host | IP Address | Interface | Network | Protected by |
|:---|:---|:---|:---|:---|
| Airtel Router | `192.168.1.1` | — | `192.168.1.0/24` | ISP Firmware |
| Proxmox Host | `192.168.1.240` | `vmbr0` | `192.168.1.0/24` | Airtel |
| Ubuntu VM | `192.168.1.18` | `ens18` | `192.168.1.0/24` | Airtel ⚠ |
| pfSense WAN | `192.168.1.x` | `vmbr0` | `192.168.1.0/24` | Airtel |
| pfSense LAN | `192.168.10.1` | `vmbr1` | `192.168.10.0/24` | **pfSense ✅** |
| TP-Link AP | `192.168.10.2` | LAN port | `192.168.10.0/24` | **pfSense ✅** |
| WiFi Clients | `192.168.10.x` | WiFi | `192.168.10.0/24` | **pfSense ✅** |

> [!IMPORTANT]
> **The Ubuntu VM is NOT behind pfSense.** It sits on `vmbr0` and connects directly to Airtel's network. Only devices on `vmbr1` (TP-Link WiFi clients) flow through pfSense. See [06 — Network Reality Check](06-network-reality-check.md) for the full traffic breakdown.

---

## What Comes Next

The rest of the docs cover exactly how I built this, what broke along the way, and the commands that fixed it:

| Next Step | Doc |
|:---|:---|
| Why I couldn't just use the Airtel router | → [02 — The ISP Wall](02-the-isp-wall.md) |
| The harsh reality of CGNAT in India | → [03 — The Final Verdict](03-the-final-verdict.md) |
