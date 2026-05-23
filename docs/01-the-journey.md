# 📖 The Journey: From "What's a Public IP?" to Enterprise Networking

> This is the complete, unfiltered story of how this project came to life — every confusion, every breakthrough, every "aha" moment.

---

## Phase 1: The Starting Point

It all started with a simple goal: **I have a spare PC with Proxmox, and I want to host my web apps with custom domains.**

What I had:
- A spare PC running **Proxmox VE**
- An **Ubuntu Server VM** running **Dokploy** with Next.js apps
- **Tailscale** installed for remote access
- An **Airtel FTTH** connection in India
- Zero networking knowledge beyond "plug in ethernet, internet works"

What I wanted:
- `myapp.mydomain.com` pointing to my self-hosted apps
- Proper port forwarding so the world can reach my server
- Understanding of how all this actually works

Simple enough, right? **Wrong.**

---

## Phase 2: The CGNAT Investigation

The first question everyone asks when you want to self-host: **"Do you have a real public IP?"**

I had no idea what that meant. So I learned.

```bash
# On Proxmox shell
root@pve:~# curl ifconfig.me
122.172.83.46

root@pve:~# curl ipinfo.io/ip
122.172.83.46
```

Then I logged into my Airtel router at `192.168.1.1` and compared the WAN IP.

**They matched.** 

This meant:
- ✅ I have a real public IPv4 address
- ✅ I'm NOT behind CGNAT
- ✅ Port forwarding should theoretically work
- ✅ Airtel uses PPPoE — good sign for real public IPs

I felt like I was 90% there. *I was not.*

---

## Phase 3: Hitting the Airtel Router Wall

### Attempt 1: Forward port 8006 (Proxmox)

I set up a port forwarding rule on my Airtel router:

| Field | Value |
|-------|-------|
| External Port | 8006 |
| Internal IP | 192.168.1.240 |
| Internal Port | 8006 |
| Protocol | TCP |

Tested from mobile data (not WiFi — learned the hard way about NAT loopback):

```
https://122.172.83.46:8006 → TIMEOUT ❌
```

But Proxmox was definitely listening:

```bash
root@pve:~# ss -tulnp | grep 8006
tcp   LISTEN 0   4096   *:8006   *:*   users:(("pveproxy worker"...))
```

Firewall was OFF. Service was running. **The problem was NOT inside Proxmox.**

### Attempt 2: Try different external port

Changed rule to External 4443 → Internal 8006. Still didn't work reliably.

### The Real Blocker: Port 443

When I tried to forward the ports I actually needed (80 and 443 for web traffic):

```
WAN 443 → Ubuntu 443
```

The router gave me:

> **"Rule conflict with Remote MGMT setting"**

**Airtel's router firmware reserves WAN port 443 for its own remote management interface.** You can't disable it. You can't override it. It's hard-coded into the ISP firmware.

This was the moment everything changed.

I tried finding hidden admin pages:
- `http://192.168.1.1/management.asp` — nope
- `http://192.168.1.1/remote.asp` — nope  
- `http://192.168.1.1/security_remote.asp` — nope

**The ISP router was the enemy.** And I couldn't even fight it.

---

## Phase 4: The Enterprise Mindset Shift

Frustrated, I asked the question that changed everything:

> *"What do enterprise people do at datacenters? Or even some homelabers?"*

The answer was eye-opening:

**Enterprise setups almost NEVER do raw port forwarding.** They use:

1. **Reverse Proxies** — Only expose 443, route everything internally via NGINX/Traefik/HAProxy
2. **VPN-first access** — Admin panels NEVER exposed publicly. Always behind VPN.
3. **Zero Trust** — Even VPNs are being replaced with identity-based access (Cloudflare Zero Trust, Google BeyondCorp)
4. **Proper firewalls** — pfSense, OPNsense, or hardware firewalls controlling everything

I learned about the 3 levels of homelab setups:

| Level | Setup | What I Was |
|-------|-------|-----------|
| 🥇 Level 1 | Tailscale/VPN-only access | Already doing this accidentally! |
| 🥈 Level 2 | Reverse proxy + single public entry | Where I wanted to be |
| 🥉 Level 3 | WAF + Firewall + Reverse Proxy + VPN | The dream |

And then the realization hit:

> *"I have Proxmox. I can install pfSense as a VM. I can have FULL control. No more Airtel firmware nonsense."*

---

## Phase 5: Planning the pfSense Setup

### The One-Port Problem

YouTube tutorials all showed pfSense with **two ethernet cables** — one for WAN, one for LAN. I only had **one ethernet port** on my spare PC.

This confused me deeply until I understood the key insight:

> **In virtualization, you can create virtual networks. LAN doesn't need a physical cable — it can be a virtual bridge inside Proxmox.**

So:
- `vmbr0` (real ethernet) = WAN (connects to Airtel router)
- `vmbr1` (virtual bridge, no physical port) = LAN (internal network)

I created `vmbr1` with no IP, no gateway, no bridge ports. This was completely safe — couldn't break anything.

### pfSense VM Creation

Downloaded pfSense CE ISO and created the VM:

| Setting | Value |
|---------|-------|
| OS | Other |
| BIOS | OVMF (UEFI) |
| Machine | q35 |
| CPU | 2 cores |
| RAM | 4 GB |
| Disk | 20 GB |
| NIC 1 (WAN) | VirtIO → vmbr0 |
| NIC 2 (LAN) | VirtIO → vmbr1 |

At this point, pfSense was a "lab router" — only virtual machines behind it, real network untouched.

---

## Phase 6: The USB Adapter Saga

### The Game Changer

I casually mentioned: *"Hey, I actually have a USB-to-Ethernet adapter lying around."*

This changed EVERYTHING. With two network interfaces, pfSense could become a **real router/firewall**, not just a lab experiment.

| Interface | Purpose |
|-----------|---------|
| Built-in Ethernet (nic0) | WAN — connects to Airtel router |
| USB Ethernet Adapter | LAN — connects to TP-Link AP |

### The Defective Adapter

Plugged it in. Checked:

```bash
root@pve:~# lsusb
Bus 004 Device 002: ID 0bda:8153 Realtek Semiconductor Corp. RTL8153 Gigabit Ethernet Adapter

root@pve:~# ip a
3: enx00e04c314578: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN
```

**State DOWN.** Tried bringing it up:

```bash
ip link set enx00e04c314578 up
```

Still wouldn't establish a link. Connected to TP-Link router — all LAN ports showing "unplugged."

After hours of debugging:
- Tried different LAN ports (LAN1, LAN2, LAN3, LAN4)
- Checked USB adapter LEDs — no lights at all
- Verified cables were good

**The USB adapter was straight up defective.** Non-functional hardware.

### The Quick Commerce Save

Had to order a **new USB-to-Ethernet adapter** via quick commerce (gotta love India's instant delivery). New adapter arrived, plugged it in — **lights on, link detected, everything working.**

---

## Phase 7: Making It All Work

### The TP-Link AP Mode Drama

Connected the new USB adapter to TP-Link router. But TP-Link's built-in "AP Mode" was being weird — disabling LAN port functionality.

**Fix:** Instead of using the built-in AP mode, manually configured it:
1. Kept router in normal mode
2. Disabled DHCP manually
3. Changed LAN IP to `10.27.27.2`
4. Connected via LAN-to-LAN (NOT WAN port)

### The Victory Moment

Checked my phone connected to TP-Link WiFi:

| Setting | Value |
|---------|-------|
| IP Address | 10.27.27.x |
| Gateway | 10.27.27.1 |
| Internet | ✅ WORKS |

**I was behind pfSense.** The firewall was handling all traffic. The architecture was real.

Opened browser:

```
https://10.27.27.1 → pfSense login page! 🎉
```

---

## Phase 8: Configuration & Victory Lap

### Setup Wizard

- Hostname: `pfsense`
- Domain: `home.arpa`
- DNS: `1.1.1.1`, `8.8.8.8`
- Timezone: `Asia/Kolkata`
- WAN: DHCP (from Airtel router)
- LAN: `10.27.27.1/24`
- Changed default password immediately

### Critical Post-Install Settings

1. **Disabled hardware checksum offloading** — essential for VM performance
2. **Verified DNS Resolver** was active
3. **Updated pfSense** to latest version

### The Snapshot

The most important step:

```
Proxmox → VM → Snapshots → Take Snapshot → "working-network"
```

A safety net. If anything breaks, roll back instantly.

---

## Looking Back

What started as *"I just want my domain to work"* became a deep dive into:

- How the internet actually works
- Why ISP routers are terrible
- How enterprises handle networking
- How to build a proper firewall architecture
- How virtual networking works in Proxmox
- Why hardware matters (and can fail on you)

The bridge confusion, WAN/LAN confusion, AP mode confusion, DHCP confusion, virtual NIC confusion — **that's the hard part of networking.** And I got through it.

Now the fun stuff starts.

---

*Total time: ~1 week of learning, debugging, and two USB ethernet adapters later* 🚀
