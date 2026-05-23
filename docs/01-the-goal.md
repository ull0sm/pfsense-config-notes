# 01. The Goal & Architecture

> This project documents my homelab attempt to understand networking by building a pfSense router. It's not a corporate enterprise setup—it's a raw look at what happens when you try to host your own apps and your ISP says "no."

---

## What I Wanted To Do

It started with a very simple goal: **I have a spare PC with Proxmox, and I want to host my web apps with custom domains.**

What I had:
- A spare PC running **Proxmox VE**
- An **Ubuntu Server VM** running **Dokploy** with Next.js apps
- **Tailscale** installed for remote access
- An **Airtel FTTH** connection in India
- Zero networking knowledge beyond "plug in ethernet, internet works"

What I wanted:
- `myapp.mydomain.com` pointing to my self-hosted apps
- Proper port forwarding so the world can reach my server

Simple enough, right? **Wrong.**

---

## The Final Architecture

After a week of debugging, breaking things, and fighting my ISP router, I ended up virtualizing a **pfSense** firewall inside Proxmox. 

This is what the final, working architecture looks like:

```mermaid
graph TD
    classDef internet fill:#2d3436,stroke:#dfe6e9,stroke-width:2px,color:#fff
    classDef isp fill:#0984e3,stroke:#74b9ff,stroke-width:2px,color:#fff
    classDef proxmox fill:#00b894,stroke:#55efc4,stroke-width:2px,color:#fff
    classDef firewall fill:#d63031,stroke:#ff7675,stroke-width:2px,color:#fff
    classDef wifi fill:#e17055,stroke:#fab1a0,stroke-width:2px,color:#fff
    classDef devices fill:#6c5ce7,stroke:#a29bfe,stroke-width:2px,color:#fff

    A(("🌐 Internet")):::internet -->|"Public IP (PPPoE)"| B["📡 Airtel ISP Router"]:::isp
    B -->|"LAN Cable"| C["🖥️ Proxmox Host<br/>(Built-in Ethernet)"]:::proxmox
    
    subgraph Proxmox Virtualized Environment
        C --> D["🔥 pfSense VM<br/>(WAN Interface - vmbr0)"]:::firewall
        D --> E["🔥 pfSense VM<br/>(LAN Interface - vmbr1)"]:::firewall
    end
    
    E -->|"USB Ethernet Adapter"| F["📶 TP-Link Router<br/>(AP Mode)"]:::wifi
    
    F -.-> G("📱 Phones"):::devices
    F -.-> H("💻 Laptops"):::devices
```

### Physical Cable Layout
```
Internet
   │
Airtel Router
   │
LAN cable
   │
Proxmox Built-in Ethernet ──── vmbr0 (WAN)
(pfSense WAN)

USB Ethernet Adapter ────────── vmbr1 (LAN)  
(pfSense LAN)
   │
LAN cable
   │
TP-Link Router LAN port (AP mode, DHCP disabled)
```

The rest of these documents cover exactly how I got here, what broke along the way, and the commands I used to fix it.
