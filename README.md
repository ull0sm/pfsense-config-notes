# 🏠 Homelab Network Documentation

> **A raw, technical documentation of my attempt to host web apps by building a pfSense router in my homelab.**
>
> This project was built to bypass my ISP's limitations. No fluff — just the exact networking steps, the pfSense setup, and the ultimate CGNAT roadblock I hit along the way.

![Status](https://img.shields.io/badge/status-working-brightgreen?style=flat-square)
![ISP](https://img.shields.io/badge/ISP-Airtel%20FTTH-blue?style=flat-square)
![Firewall](https://img.shields.io/badge/firewall-pfSense%20CE-red?style=flat-square)
![Hypervisor](https://img.shields.io/badge/hypervisor-Proxmox%20VE-orange?style=flat-square)
![Location](https://img.shields.io/badge/location-India-white?style=flat-square)

---

## 📖 The Short Version

I wanted to self-host web apps on a spare PC. My ISP router said **no** (literally blocked port 443). So I learned how to virtualize my own firewall from scratch.

```mermaid
flowchart LR
    A(("🌐\nInternet")) --> B["📡 Airtel Router\n192.168.1.1"]
    B -->|"LAN cable"| C["🖥️ Proxmox VE\n192.168.1.240"]
    C --> D["🔥 pfSense VM\nWAN: 192.168.1.x\nLAN: 192.168.10.1"]
    D -->|"USB Ethernet\nenxXXXXXXXXXXXX"| E["📶 TP-Link AP\nDHCP disabled"]
    E -.->|"WiFi"| F(("📱 Protected\nDevices\n192.168.10.x"))

    style A fill:#2d3436,stroke:#dfe6e9,color:#fff
    style B fill:#0984e3,stroke:#74b9ff,color:#fff
    style C fill:#00b894,stroke:#55efc4,color:#fff
    style D fill:#d63031,stroke:#ff7675,color:#fff
    style E fill:#e17055,stroke:#fab1a0,color:#fff
    style F fill:#6c5ce7,stroke:#a29bfe,color:#fff
```

---

## 📚 The War Stories

| # | Doc | What It Covers |
|:---:|:---|:---|
| **01** | [🗺️ The Goal & Architecture](docs/01-the-goal.md) | Full physical + logical network diagrams, real IPs, bridge mapping, and the two-network split. |
| **02** | [🧱 The ISP Wall](docs/02-the-isp-wall.md) | CGNAT check, NAT loopback trap, and the port 443 firmware lockout that forced pfSense. |
| **03** | [🛑 The Final Verdict](docs/03-the-final-verdict.md) | The CGNAT realization, static IP costs in India, and why self-hosting on Airtel FTTH hit a dead end. |

---

## 🛠️ Tech Stack

| Layer | Tool |
|:---|:---|
| **Hypervisor** | Proxmox VE |
| **Firewall / Router** | pfSense CE (Virtualized) |
| **Virtual Networking** | VirtIO Bridges — `vmbr0`, `vmbr1` |
| **Second NIC** | USB-to-Ethernet (`enxXXXXXXXXXXXX`, RTL8153) |

---

## 🌍 Context

- **Location:** India 🇮🇳
- **ISP:** Airtel FTTH — PPPoE connection (Initial thought: real public IPv4. Reality: CGNAT trap)
- **Goal:** Understand how traffic routes from the internet to internal services by owning the firewall layer myself

---

*This is a living document of my homelab learning process. The struggle is the documentation.* 🔧
