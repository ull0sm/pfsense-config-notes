# 04. The Escape Plan — VPS + Cloudflare Proxy

> **TL;DR:** Direct port forwarding is dead. Cloudflare Tunnel is off the table. After researching every option, the real answer is a two-part stack: a cheap VPS punches through CGNAT, and Cloudflare's free proxy layer sits in front of it. Together they form a proper production-grade setup. The app never leaves home.

---

## The Situation After Doc 03

By the end of [03 — The Final Verdict](03-the-final-verdict.md), I had:

- A fully working pfSense VM inside Proxmox ✅
- A real understanding of Linux network bridges ✅
- Absolutely zero ability to receive inbound traffic from the internet ❌

The wall wasn't the firewall config. It was the ISP itself.

| What I tried | Result |
|:---|:---|
| Port forward 443 on Airtel router | Firmware hard-blocks it |
| pfSense VM to bypass Airtel firmware | Still behind CGNAT — no public IPv4 |
| Ask Airtel to remove CGNAT | They offered Static IP for ₹350/month |
| Static IP from Airtel | ₹350/mo + unreliable field engineers + ISP dependency forever |
| Cloudflare Tunnel | Not doing it — I want to understand this properly |

One option left.

---

## The Solution — Two Parts, One Stack

These are not alternatives. They solve different problems. Both are needed.

### Part 1 — VPS Reverse Proxy (kills the CGNAT problem)

Your home server can't receive inbound connections. But it can make outbound ones. So you rent a cheap VPS with a real public IP, punch a WireGuard tunnel outward from home to the VPS, and the VPS forwards incoming traffic back through that tunnel.

```
CGNAT blocks:  inbound  ❌  (nobody can call you)
CGNAT allows:  outbound ✅  (you can call others)
```

The VPS is not hosting your app. It's just a **public doorway** — the one thing your home server can't be.

### Part 2 — Cloudflare Proxy (free layer on top)

> [!NOTE]
> **This is NOT Cloudflare Tunnel.** These are completely different things.

| | Cloudflare **Tunnel** | Cloudflare **Proxy** |
|:---|:---|:---|
| What it is | A daemon you install that routes traffic through Cloudflare | DNS proxy — orange cloud on your A record |
| Requires installing anything | ✅ Yes (`cloudflared` daemon on your machine) | ❌ No — just a DNS toggle |
| Solves CGNAT by itself | ✅ Yes | ❌ No — still needs a public IP |
| Hides your VPS IP from internet | ❌ No | ✅ Yes |
| Free | ✅ | ✅ |
| Are we doing this | ❌ | ✅ |

Cloudflare Proxy alone can't solve CGNAT — it still needs something to forward traffic to. But once the VPS has a public IP, Cloudflare Proxy is a **free security and performance upgrade** on top.

---

## The Full Stack

```
Internet → Cloudflare Proxy → VPS Nginx → WireGuard tunnel → Home Proxmox → Dokploy → App
```

---

## What Each Layer Does

```mermaid
flowchart TD
    USER(("🌍 Internet User"))

    CF["🟠 Cloudflare Proxy
    ─────────────────────
    FREE
    ─────────────────────
    Hides VPS IP from internet
    Free SSL / TLS
    Free DDoS protection
    CDN + caching
    Bot filtering"]

    VPS["☁️ VPS
    ─────────────────────
    ~₹300-400/month
    ─────────────────────
    Public IPv4
    Nginx reverse proxy
    WireGuard server
    UFW firewall"]

    TUNNEL{{"🔐 WireGuard Tunnel
    Encrypted
    Initiated outbound from home
    Invisible to CGNAT"}}

    HOME["🏠 Home Proxmox
    ─────────────────────
    Behind CGNAT — invisible
    ─────────────────────
    WireGuard client
    Dokploy
    Your app container"]

    USER -->|"HTTPS request"| CF
    CF -->|"proxies to VPS IP\n(Cloudflare ranges only)"| VPS
    VPS -->|"forwards via tunnel"| TUNNEL
    TUNNEL -->|"reaches app"| HOME
    HOME -->|"response back\nsame path"| TUNNEL
    TUNNEL --> VPS --> CF --> USER

    style USER fill:#2d3436,stroke:#dfe6e9,color:#fff
    style CF fill:#f39c12,stroke:#f1c40f,color:#fff,font-weight:bold
    style VPS fill:#0984e3,stroke:#74b9ff,color:#fff,font-weight:bold
    style TUNNEL fill:#6c5ce7,stroke:#a29bfe,color:#fff
    style HOME fill:#00b894,stroke:#55efc4,color:#fff,font-weight:bold
```

---

## Full Architecture Diagram

```mermaid
flowchart LR
    subgraph PUBLIC ["🌍 What the Internet Sees"]
        USER(("User\nbrowser"))
        CF_EDGE["🟠 Cloudflare Edge
        ─────────────────
        Anycast IP
        Free SSL
        DDoS protection
        CDN cache"]
    end

    subgraph VPS_ZONE ["☁️ VPS (~₹300-400/mo)"]
        VPS_BOX["Nginx / Caddy
        reverse proxy
        ─────────────────
        WireGuard Server
        10.0.0.1
        ─────────────────
        UFW: allow CF IPs only
        + WG port"]
    end

    subgraph TUNNEL_ZONE ["🔐 WireGuard Tunnel"]
        WG_LINK{{"10.0.0.1 ↔ 10.0.0.2
        Encrypted
        Outbound from home"}}
    end

    subgraph HOME ["🏠 Home — Behind CGNAT — Invisible"]
        PVE["Proxmox Host
        ─────────────────
        WireGuard Client
        10.0.0.2
        ─────────────────
        Dokploy
        ─────────────────
        App Container
        :3000 / :8080"]
    end

    USER -->|"HTTPS"| CF_EDGE
    CF_EDGE -->|"proxy to VPS IP"| VPS_BOX
    VPS_BOX <-->|" "| WG_LINK
    WG_LINK <-->|"outbound\nfrom home"| PVE

    style PUBLIC fill:#fff8e8,stroke:#f39c12,color:#2d3436
    style VPS_ZONE fill:#eaf4ff,stroke:#0984e3,color:#2d3436
    style TUNNEL_ZONE fill:#f5f0ff,stroke:#6c5ce7,color:#2d3436
    style HOME fill:#f0fff4,stroke:#00b894,color:#2d3436
    style CF_EDGE fill:#f39c12,stroke:#e67e22,color:#fff,font-weight:bold
    style VPS_BOX fill:#0984e3,stroke:#74b9ff,color:#fff
    style PVE fill:#00b894,stroke:#55efc4,color:#fff
```

---

## Where Does Everything Live?

| Component | Lives where | Visible to internet? |
|:---|:---|:---|
| Your web app | 🏠 Home Proxmox — Dokploy | ❌ No — only reachable via tunnel |
| WireGuard client | 🏠 Home Proxmox | ❌ No — outbound only |
| Nginx / Caddy | ☁️ VPS | Only to Cloudflare IPs |
| WireGuard server | ☁️ VPS | UDP port only (51820) |
| Cloudflare Proxy | 🟠 Cloudflare edge | ✅ Yes — this is what the world hits |
| Your domain DNS | 🌐 Cloudflare | `A record → VPS IP`, orange cloud on |

> [!IMPORTANT]
> **The app never moves to the VPS.** The VPS never exposes itself directly. Cloudflare is the only thing the world ever sees.

---

## What You Install Where

### 🏠 On Proxmox (home)

You probably have most of this already:

```
Proxmox VE
└── Dokploy            ← running your app containers  ✅
└── WireGuard client   ← new, connects outbound to VPS
```

Nothing faces the internet. No ports opened on your home router.

---

### ☁️ On the VPS (fresh install)

```
VPS (Ubuntu 22.04 recommended)
├── WireGuard server    → listens on UDP 51820
├── Nginx / Caddy       → listens on 80, 443 (Cloudflare IPs only)
├── UFW firewall        → blocks everything except the above
└── fail2ban            → SSH brute force protection
```

---

### 🟠 On Cloudflare DNS

```
A     app.yourdomain.com     →    VPS_PUBLIC_IP     🟠 Proxy ON
```

That's it. One record. Orange cloud enabled. No daemon. No agent. Nothing installed.

---

## Security Model

Three independent layers. Each one blocks a different class of attack.

```mermaid
flowchart TD
    subgraph L1 ["🟠 Layer 1 — Cloudflare (Free)"]
        CF["Hides VPS IP from internet
        Absorbs DDoS before it hits VPS
        Filters bots and bad actors
        Free SSL termination"]
    end

    subgraph L2 ["☁️ Layer 2 — VPS"]
        VPS["UFW: only Cloudflare IP ranges on 80/443
        WireGuard UDP port open (51820)
        SSH: key-only, no passwords
        fail2ban active"]
    end

    subgraph L3 ["🏠 Layer 3 — Home (invisible)"]
        HOME["Not reachable from internet at all
        Proxmox UI (8006) — never forwarded
        Dokploy ports — internal only
        WireGuard routes only app traffic"]
    end

    L1 -->|"only clean traffic\npasses through"| L2
    L2 -->|"only encrypted\ntunnel"| L3

    style L1 fill:#fff8e8,stroke:#f39c12,color:#2d3436
    style L2 fill:#eaf4ff,stroke:#0984e3,color:#2d3436
    style L3 fill:#f0fff4,stroke:#00b894,color:#2d3436
```

**VPS hardening checklist:**

- [ ] Disable SSH password login — use keys only
- [ ] UFW: allow only [Cloudflare IP ranges](https://www.cloudflare.com/ips/) on ports 80 and 443
- [ ] UFW: allow `51820/udp` (WireGuard)
- [ ] UFW: allow `22` (SSH — lock to your IP if possible)
- [ ] Install `fail2ban`
- [ ] Enable unattended security updates

**What NOT to do:**

> [!CAUTION]
> - ❌ Don't port forward Proxmox UI port `8006` from your home router
> - ❌ Don't open Dokploy or Docker ports publicly
> - ❌ Don't leave VPS ports 80/443 open to all IPs — only Cloudflare ranges
> - ❌ Don't route your entire home LAN through WireGuard — only the app tunnel

---

## How This Compares to Everything Else

| Option | Solves CGNAT | Home IP hidden | VPS IP hidden | No CF Tunnel | Monthly cost |
|:---|:---|:---|:---|:---|:---|
| Direct port forward | ❌ | ❌ | — | ✅ | Free |
| Cloudflare Tunnel | ✅ | ✅ | ✅ | ❌ | Free |
| Static IP from Airtel | ✅ | ❌ | — | ✅ | ₹350/mo |
| VPS only | ✅ | ✅ | ❌ | ✅ | ₹300-400/mo |
| **VPS + Cloudflare Proxy** | **✅** | **✅** | **✅** | **✅** | **₹300-400/mo** |

---

## The Restaurant Analogy

> 🏠 Your home = the kitchen. Food is made here. Nobody enters.  
> ☁️ VPS = the restaurant counter. Takes orders from Cloudflare, gets food from the kitchen.  
> 🟠 Cloudflare = the bouncer at the door. Filters who even gets to talk to the counter.  
> 🌍 Internet = customers outside.
>
> Customers talk to the bouncer. Clean traffic reaches the counter. The counter calls the kitchen. Nobody ever sees the kitchen.

---

## What Comes Next

This doc covers the architecture and the reasoning. The actual setup follows.

| Next Step | Doc |
|:---|:---|
| WireGuard server + client setup | → [05 — VPS Setup](05-vps-setup.md) *(coming soon)* |
| Nginx reverse proxy config on VPS | → [05 — VPS Setup](05-vps-setup.md) *(same doc)* |
| Cloudflare DNS + SSL/TLS config | → [05 — VPS Setup](05-vps-setup.md) *(final section)* |
| Dokploy internal routing | → [06 — Dokploy Routing](06-dokploy-routing.md) *(coming soon)* |

> *I spent a week virtualizing a firewall to hit a concrete CGNAT wall. Then I found out two tools — a ₹300/mo VPS and Cloudflare's free proxy — solve the whole thing when used together. One handles the ISP problem. The other handles the internet problem. Together they're a proper stack.*