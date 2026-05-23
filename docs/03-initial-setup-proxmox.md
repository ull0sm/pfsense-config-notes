# 🖥️ Initial Setup: Proxmox Foundation

> What was already in place before the pfSense project began.

---

## Hardware

| Component | Details |
|-----------|---------|
| Machine | Spare PC repurposed as homelab server |
| Ethernet | Single built-in Realtek NIC (`nic0`) |
| MAC | `00:e0:4c:58:fa:bf` |
| WiFi Dongle | TP-Link AC1300 Archer T3U Plus USB 3.0 (NOT used for networking) |
| USB Adapter | USB-to-Ethernet (Realtek RTL8153) — added later |

---

## Proxmox Configuration

| Setting | Value |
|---------|-------|
| Platform | Proxmox VE |
| Host IP | `192.168.1.240/24` |
| Gateway | `192.168.1.1` |
| Bridge | `vmbr0` (bridge port: `nic0`) |
| Web UI | `https://192.168.1.240:8006` |
| Firewall | Disabled at datacenter level |

### Network Interfaces (from `ip a`)

```
1: lo          — loopback (127.0.0.1)
2: nic0        — built-in ethernet, master vmbr0, state UP
3: wlx74fe...  — WiFi dongle, state DOWN (unused)
4: tailscale0  — 100.92.142.123/32 (Tailscale VPN)
5: vmbr0       — 192.168.1.240/24 (main bridge)
7: tap100i0    — VM tap device
8: fwbr100i0   — firewall bridge
9: fwpr100p0   — firewall proxy
10: fwln100i0  — firewall link
```

---

## VMs Running

### Ubuntu Server (ID: 100)

| Setting | Value |
|---------|-------|
| Hostname | `vandahouse` |
| IP | `192.168.1.12/24` |
| Tailscale IP | `100.107.0.40` |
| IPv6 | `2401:4900:8813:9a95:be24:11ff:fe6f:85ff` |

**Running Services:**
- **Dokploy** — deployment platform for web apps
- **Docker** with multiple containers
- **Traefik** reverse proxy (via Dokploy)
- **Next.js applications**
- Ports 80 and 443 listening

```bash
nano@vandahouse:~$ ss -tulnp | grep :80
tcp   LISTEN 0   4096   0.0.0.0:80    0.0.0.0:*
tcp   LISTEN 0   4096      [::]:80       [::]:*

nano@vandahouse:~$ ss -tulnp | grep :443
tcp   LISTEN 0   4096   0.0.0.0:443   0.0.0.0:*
tcp   LISTEN 0   4096      [::]:443      [::]:*
```

---

## Remote Access

| Method | Details |
|--------|---------|
| **Tailscale** | `https://100.92.142.123:8006` — primary remote access |
| **Cloudflare Tunnel** | Also configured as backup |
| **Local** | `https://192.168.1.240:8006` from home network |

---

## ISP Details

| Detail | Value |
|--------|-------|
| ISP | Airtel (India) |
| Connection | FTTH (Fiber to the Home) |
| WAN Type | PPPoE |
| Public IP | `122.172.83.46` (dynamic) |
| CGNAT | **No** — confirmed real public IPv4 |
| Router | ISP-provided Airtel router/ONT combo |
| Router Admin | `http://192.168.1.1` |
| Router WAN Interface | `PPPoE-Internet` |

---

## Network Topology (Pre-pfSense)

```
Internet
   │
Airtel Router (192.168.1.1)
   │ WiFi + LAN
   │
   ├── Phones, Laptops, TVs (via WiFi)
   │
   └── Proxmox PC (192.168.1.240 via Ethernet)
          │
          ├── Ubuntu VM (192.168.1.12)
          │     └── Dokploy → Next.js apps
          │
          └── Tailscale (100.92.142.123)
```

This was the starting point. Everything worked internally. The challenge was exposing services to the internet — which led to the pfSense project.
