# 🔍 06. Network Reality Check — Who Is Actually Behind pfSense?

> **TL;DR:** Not everything on Proxmox is behind pfSense. The Ubuntu VM talks directly to Airtel. Only TP-Link WiFi clients go through pfSense. This doc explains exactly why, and what it means.

---

## 🗺️ The Two Networks

Proxmox has two virtual bridges, and they are **completely separate networks:**

```mermaid
flowchart LR
    subgraph AIRTEL ["☁️ vmbr0 — Airtel Network (192.168.1.0/24)"]
        direction TB
        A_GW["📡 Airtel Router\n192.168.1.1\n(your gateway)"]
        A_PVE["🖥️ Proxmox Host\n192.168.1.240"]
        A_UBU["🐧 Ubuntu VM\n192.168.1.18\nens18"]
        A_PFS["🔥 pfSense WAN NIC\n192.168.1.x"]
    end

    subgraph PFSENSE ["🔐 vmbr1 — pfSense LAN (192.168.10.0/24)"]
        direction TB
        P_GW["🔥 pfSense LAN\n192.168.10.1\n(your gateway)"]
        P_TP["📶 TP-Link AP\n192.168.10.2"]
        P_DEV1(("💻 Laptop\n192.168.10.x"))
        P_DEV2(("📱 Phone\n192.168.10.x"))
    end

    style AIRTEL fill:#fff3e0,stroke:#e17055,color:#2d3436
    style PFSENSE fill:#e8f5e9,stroke:#00b894,color:#2d3436
    style A_UBU fill:#fdcb6e,stroke:#e17055,color:#2d3436,font-weight:bold
    style P_GW fill:#d63031,stroke:#ff7675,color:#fff
```

| Bridge | NIC | Subnet | Firewall |
|:---|:---|:---|:---|
| `vmbr0` | Built-in Ethernet | `192.168.1.0/24` | Airtel Router (ISP firmware) |
| `vmbr1` | USB Ethernet `enxXXXXXXXXXXXX` | `192.168.10.0/24` | **pfSense** ✅ |

---

## 📦 Traffic Flow 1 — Ubuntu VM (Bypasses pfSense)

```mermaid
sequenceDiagram
    participant DEV as 📱 External Device
    participant INET as 🌐 Internet
    participant AIR as 📡 Airtel Router
    participant UBU as 🐧 Ubuntu VM<br/>192.168.1.18

    DEV->>INET: Request to your app
    INET->>AIR: Hits your public IP
    AIR->>UBU: Port forward → Ubuntu (vmbr0)
    Note over AIR,UBU: pfSense is completely skipped
    UBU->>INET: Response goes back
```

> [!WARNING]
> **Ubuntu is NOT behind pfSense.** It sits directly on `vmbr0` — the same network segment as the Airtel router. Its traffic never passes through pfSense's firewall rules. The only firewall protecting it is Airtel's consumer-grade firmware.

**What this means in practice:**

| Security question | Answer |
|:---|:---|
| Does pfSense protect Ubuntu? | ❌ No |
| Who firewalls Ubuntu? | Airtel's router (consumer firmware) |
| Can pfSense LAN reach Ubuntu? | ✅ Yes, via routing through Airtel |
| Can I apply pfSense rules to Ubuntu? | ❌ Not without moving it to `vmbr1` |

---

## 🔐 Traffic Flow 2 — TP-Link WiFi Clients (Behind pfSense)

```mermaid
sequenceDiagram
    participant DEV as 📱 External Device
    participant INET as 🌐 Internet
    participant AIR as 📡 Airtel Router
    participant PF as 🔥 pfSense
    participant TP as 📶 TP-Link AP
    participant CLI as 💻 WiFi Client<br/>192.168.10.x

    DEV->>INET: Request
    INET->>AIR: Hits public IP
    AIR->>PF: Forward to pfSense WAN
    Note over PF: Evaluate firewall rules
    PF->>TP: Pass through LAN (vmbr1)
    TP->>CLI: WiFi → Client
    CLI->>PF: Response
    PF->>AIR: NAT + route out
    AIR->>INET: Back to internet
```

**What this means in practice:**

| Question | Answer |
|:---|:---|
| Does pfSense protect WiFi clients? | ✅ Yes, fully |
| Who handles DHCP for WiFi clients? | pfSense (gives `192.168.10.x`) |
| Who handles DNS for WiFi clients? | pfSense DNS Resolver |
| Can I write firewall rules for WiFi clients? | ✅ Yes, in pfSense |

---

## 🧠 Why the Split Exists

pfSense needs two interfaces to function as a firewall:

```mermaid
flowchart TD
    NEED["pfSense needs\ntwo interfaces"]

    NEED --> WAN & LAN

    WAN["WAN Interface
    Connected to vmbr0
    Gets IP from Airtel DHCP
    (192.168.1.x)
    —
    Traffic from outside\ncomes in here"]

    LAN["LAN Interface
    Connected to vmbr1
    Serves as gateway for\ninternal network
    (192.168.10.1)
    —
    Traffic from devices\ngoes through here"]

    WAN -->|"pfSense routes\n+ applies rules"| LAN

    style WAN fill:#0984e3,stroke:#74b9ff,color:#fff
    style LAN fill:#00b894,stroke:#55efc4,color:#fff
    style NEED fill:#6c5ce7,stroke:#a29bfe,color:#fff
```

The Ubuntu VM was intentionally kept on `vmbr0` because:
- It needs direct internet access for Dokploy deployments
- Tailscale handles its remote access security
- Moving it to `vmbr1` would require updating firewall rules, DNS, and port forwards

---

## 🔧 How to Move Ubuntu Behind pfSense (Future)

If you ever want Ubuntu behind pfSense, here's what changes:

| Step | Action |
|:---|:---|
| 1 | In Proxmox, change Ubuntu VM's network device from `vmbr0` → `vmbr1` |
| 2 | Ubuntu will get a `192.168.10.x` IP from pfSense DHCP |
| 3 | Update pfSense port forward rules to point to the new IP |
| 4 | Update any DNS records pointing to Ubuntu |
| 5 | Remove the old `192.168.1.18` static rule from Airtel router |

> [!NOTE]
> After moving, all Ubuntu traffic will flow through pfSense firewall rules. This is more secure but requires you to explicitly allow the traffic you want.

---

## 📋 Final Summary

| Host | IP | Protected By | Behind pfSense? |
|:---|:---|:---|:---|
| Airtel Router | `192.168.1.1` | ISP | — |
| Proxmox Host | `192.168.1.240` | Airtel | ❌ |
| Ubuntu VM | `192.168.1.18` | Airtel | ❌ |
| pfSense WAN | `192.168.1.x` | Airtel | — |
| pfSense LAN | `192.168.10.1` | pfSense | — (is pfSense) |
| TP-Link AP | `192.168.10.2` | pfSense | ✅ |
| WiFi Clients | `192.168.10.x` | pfSense | ✅ |
