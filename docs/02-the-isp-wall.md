# 02. The ISP Wall

> Before you can host a website, you need a public IP. This documents the investigation into CGNAT and the exact moment the Airtel router proved it was unfit for the job.

---

## What I Tried: The CGNAT Investigation

The first question everyone asks when you want to self-host: **"Do you have a real public IP?"**

I had no idea what that meant, so I ran a check from the Proxmox shell:

```bash
# On Proxmox shell
root@pve:~# curl ifconfig.me
122.172.83.46

root@pve:~# curl ipinfo.io/ip
122.172.83.46
```

I logged into my Airtel router at `192.168.1.1` and checked the WAN IP. **They matched.** 

This meant:
- ✅ I have a real public IPv4 address.
- ✅ I'm NOT behind CGNAT.
- ✅ Port forwarding should theoretically work.

I felt like I was 90% there. *I was not.*

---

## What Failed: The Port Forwarding Struggle

### Attempt 1: Forward port 8006 (Proxmox)
I set up a port forwarding rule on the Airtel router (`8006 → 192.168.1.240:8006`).

**Symptom:** Tested from my phone (on WiFi) → Timeout.

**Investigation:**
```bash
# Confirmed service is running
root@pve:~# ss -tulnp | grep 8006
tcp   LISTEN 0   4096   *:8006   *:*   users:(("pveproxy worker"...))

# Confirmed firewall is OFF in Proxmox UI: Datacenter → Firewall → disabled
```

**Root Cause:**
I was testing from the *same* WiFi network. The Airtel router doesn't support **NAT loopback** (hairpinning). You cannot access your own public IP from inside the house.

**Fix:** Tested from mobile data (disconnected from WiFi).

### Attempt 2: The Port 443 Block
When I tried to forward the ports I actually needed for web traffic (`WAN 443 → Ubuntu 443`), the router threw a hard error:

> [!WARNING]
> **"Rule conflict with Remote MGMT setting"**

**Root Cause:**
Airtel's router firmware hard-reserves WAN port 443 for its own remote management interface. 

**What I Tried (And Failed):**
- Tried different port forwarding fields: Same error.
- Searched for hidden admin URLs (`/management.asp`, `/remote.asp`): `404 Not Found`.
- Tried Telnet/SSH to the router: Connection refused.
- Firmware update: ISP-locked.

---

## The Fix: Bypassing the ISP

There was no fix within the ISP router. I realized that ISP routers are designed for consumers, not for self-hosting. 

If I wanted real control—the kind of control SREs have over firewalls—I couldn't rely on Airtel's firmware. This limitation was the exact catalyst that forced me to virtualize my own router using **pfSense** in Proxmox.

> [!TIP]
> **Lesson Learned:** Port forwarding failures are rarely just "one thing." Check: ISP filtering → router rules → firewall → service → testing method (NAT loopback). If the ISP locks the port, your only option is to replace their routing with your own firewall.
