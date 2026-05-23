# 🚧 The Router Problem — Why the ISP Router Had to Go

> The investigation that sparked the entire pfSense project.

---

## The Goal

Self-host web applications (Next.js via Dokploy) with custom domain names:

```
myapp.mydomain.com → my home server → Dokploy → Next.js app
```

Everything was working internally. The Ubuntu server was listening on ports 80/443. Dokploy was running. Apps were live. **The only missing piece was exposing them to the internet.**

---

## Investigation Timeline

### 🔍 Test 1: Is this CGNAT?

```bash
root@pve:~# curl ifconfig.me
122.172.83.46
```

Compared with router WAN IP → **matched**. 

**Verdict:** NOT behind CGNAT. Real public IPv4. PPPoE confirmed.

---

### 🔍 Test 2: Port Forward Proxmox (8006)

Set up port forwarding on Airtel router:

| Field | Value |
|-------|-------|
| WAN Interface | PPPoE-Internet |
| External Port | 8006 |
| Internal IP | 192.168.1.240 |
| Internal Port | 8006 |
| Protocol | TCP |

Tested from mobile data:

```
https://122.172.83.46:8006 → TIMEOUT ❌
```

But Proxmox was definitely running:

```bash
root@pve:~# ss -tulnp | grep 8006
tcp   LISTEN 0   4096   *:8006   *:*   users:(("pveproxy worker"...))
```

Proxmox firewall: **DISABLED**. Not the problem.

---

### 🔍 Test 3: Different External Port

Changed to External 4443 → Internal 8006 (to bypass possible ISP port filtering on 8006).

**Still unreliable.** Some ISPs filter entire port ranges.

---

### 💥 Test 4: The Real Blocker — Port 443

Tried forwarding the ports actually needed for web apps:

```
WAN 443 → Ubuntu 443
```

**Router response:**

> ⛔ **"Rule conflict with Remote MGMT setting"**

**The Airtel router firmware reserves WAN port 443 for its own HTTPS remote management interface.**

This is hard-coded. Can't disable it. Can't override it.

---

## What I Tried to Fix It

### Hidden Admin Pages

Tried accessing hidden firmware pages:

| URL | Result |
|-----|--------|
| `http://192.168.1.1/management.asp` | Not found |
| `http://192.168.1.1/remote.asp` | Not found |
| `http://192.168.1.1/security_remote.asp` | Not found |
| Telnet/SSH to router | No access |

**None worked.** Airtel locks the firmware down.

### Workaround: Non-Standard Ports

| WAN Port | Ubuntu Port | URL |
|----------|-------------|-----|
| 8080 | 80 | `http://domain.com:8080` |
| 8443 | 443 | `https://domain.com:8443` |

**This works** but produces ugly URLs and breaks standard HTTPS.

### Cloudflare Tunnel

Works without any port forwarding. But adds external dependency and doesn't solve the fundamental control issue.

---

## Why ISP Routers Are The Problem

| Limitation | Impact |
|-----------|--------|
| Port 443 reserved for remote management | Can't host HTTPS services |
| Port 80 sometimes reserved too | Can't host HTTP services |
| Weak firewall controls | Can't create proper security rules |
| No VLAN support | Can't segment networks |
| Poor DNS control | Can't use custom DNS filtering |
| Bad UPnP implementation | Unreliable automatic port mapping |
| CGNAT sometimes enabled | No inbound connectivity at all |
| No bridge mode (some models) | Can't bypass the router entirely |
| Locked firmware | Can't modify any of the above |

---

## The Decision

> *"Since I have Proxmox, I can install pfSense as a VM and have FULL control over my network — no more Airtel firmware nonsense."*

The options were:

| Option | Pros | Cons |
|--------|------|------|
| Keep ISP router + workarounds | Simple, no changes | Ugly ports, limited control |
| Cloudflare Tunnel | Works behind any NAT | External dependency |
| **pfSense on Proxmox** | **Full control, enterprise-grade** | **Learning curve (worth it)** |

**Option 3 won.** Not just because it solves the port problem — but because understanding network security at this level is essential for the SRE/Platform Engineer path.

This was the moment the project shifted from *"trying to work around ISP limitations"* to *"building enterprise-grade network infrastructure."*

---

## Key Takeaway

> ISP routers are designed for consumers who just need WiFi. The moment you want to self-host, expose services, or control your network — they become your biggest bottleneck. The solution isn't to fight them. It's to replace them (or at least bypass them).
