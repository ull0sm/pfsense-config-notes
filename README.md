# Homelab Documentation
> **A raw, technical documentation of building a homelab from scratch — virtualized firewalls, self-hosting apps, fighting ISPs, and figuring out how to expose services to the internet without losing your mind.**
>
> Started as a pfSense experiment. Became a full homelab build log. No fluff — just the actual steps, the actual failures, and the fixes.

![Status](https://img.shields.io/badge/status-in%20progress-yellow?style=flat-square)
![ISP](https://img.shields.io/badge/ISP-Airtel%20FTTH-blue?style=flat-square)
![Firewall](https://img.shields.io/badge/firewall-pfSense%20CE-red?style=flat-square)
![Hypervisor](https://img.shields.io/badge/hypervisor-Proxmox%20VE-orange?style=flat-square)
![Stack](https://img.shields.io/badge/stack-Dokploy%20%7C%20WireGuard%20%7C%20Cloudflare-6c5ce7?style=flat-square)
![Location](https://img.shields.io/badge/location-India-white?style=flat-square)

---

## The Short Version

I wanted to self-host web apps on a spare PC. My ISP router blocked port 443 at the firmware level. So I virtualized my own firewall, hit a CGNAT wall, and eventually figured out the proper way to expose services from behind a consumer ISP in India.

```mermaid
flowchart LR
    Internet((Internet))
    CF["🟠 Cloudflare Proxy"]
    VPS["☁️ VPS\nNginx + WireGuard"]
    Proxmox["Proxmox VE\n192.168.1.240"]
    PFSense["pfSense VM\nWAN: 192.168.1.x\nLAN: 192.168.10.1"]
    Dokploy["Dokploy\nApp Containers"]

    Internet --> CF
    CF --> VPS
    VPS -->|"WireGuard tunnel"| Proxmox
    Proxmox --> PFSense
    Proxmox --> Dokploy
```

---

## The War Stories

| # | Doc | What It Covers |
|:---:|:---|:---|
| **01** | [The Goal & Architecture](docs/01-the-goal.md) | Full physical + logical network diagrams, real IPs, bridge mapping, and the two-network split. |
| **02** | [The ISP Wall](docs/02-the-isp-wall.md) | CGNAT check, NAT loopback trap, and the port 443 firmware lockout that forced pfSense. |
| **03** | [The Final Verdict](docs/03-the-final-verdict.md) | The CGNAT realization, static IP costs in India, and why direct hosting on Airtel FTTH is a dead end. |
| **04** | [The Escape Plan](docs/04-the-escape-plan.md) | The actual solution — VPS reverse proxy + WireGuard tunnel + Cloudflare Proxy as a production-grade stack. |

---

## Tech Stack

| Layer | Tool |
|:---|:---|
| **Hypervisor** | Proxmox VE |
| **Firewall / Router** | pfSense CE (Virtualized) |
| **Virtual Networking** | VirtIO Bridges — `vmbr0`, `vmbr1` |
| **Second NIC** | USB-to-Ethernet (`enxXXXXXXXXXXXX`, RTL8153) |
| **App Management** | Dokploy |
| **Tunnel** | WireGuard |
| **Public Proxy** | Cloudflare Proxy (free tier) |
| **VPN** | Tailscale |

---

## Context

- **Location:** India
- **ISP:** Airtel FTTH — behind CGNAT (no real public IPv4 on residential)
- **Hardware:** Spare PC running Proxmox VE as a full homelab server
- **Goal:** Understand how traffic routes from the internet to internal services by owning every layer myself

---

*This is a living build log. Every doc is a real problem I hit and actually solved. The struggle is the documentation.*