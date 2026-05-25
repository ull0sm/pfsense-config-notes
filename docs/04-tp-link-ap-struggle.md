# 📶 04. The TP-Link AP Struggle

> **TL;DR:** I needed a WiFi access point. I had a TP-Link router. I hit one magic "AP Mode" button, everything broke. The fix was ignoring that button and configuring it manually.

---

## 🎯 What I Wanted

Once pfSense was running and routing traffic through the USB ethernet adapter (`vmbr1`), I needed a way to give my devices WiFi. The plan was simple:

```mermaid
flowchart LR
    PF["🔥 pfSense LAN\n192.168.10.1"]
    TP["📶 TP-Link\n(dumb WiFi antenna)"]
    DEV(("📱 My Devices\n192.168.10.x\nIP from pfSense"))

    PF -->|"vmbr1 → USB NIC"| TP
    TP -.->|"WiFi"| DEV

    style PF fill:#d63031,stroke:#ff7675,color:#fff
    style TP fill:#e17055,stroke:#fab1a0,color:#fff
    style DEV fill:#6c5ce7,stroke:#a29bfe,color:#fff
```

The TP-Link should:
- ✅ Broadcast WiFi
- ✅ Pass DHCP requests up to pfSense
- ❌ NOT run its own DHCP
- ❌ NOT do its own NAT/routing

That's called an **Access Point**. The TP-Link has a built-in "AP Mode" toggle. Sounds perfect.

---

## ❌ What Failed

### Failure 1 — The Built-in AP Mode Toggle

I hit the "Switch to Access Point Mode" button in the TP-Link admin panel.

**The network broke immediately.**

```mermaid
flowchart TD
    TOGGLE["🔘 Enable 'AP Mode'\nin TP-Link settings"]
    TOGGLE --> S1 & S2

    S1["TP-Link shows all\nLAN ports as 'unplugged'\neven with cable connected 🤔"]
    S2["Devices get\nno IP address\nfrom pfSense ❌"]

    S1 --> ROOT["Root Cause: TP-Link's AP mode\nbuggy firmware changed internal\nswitching — isolated the pfSense\nDHCP server from WiFi clients"]
    S2 --> ROOT

    style TOGGLE fill:#0984e3,stroke:#74b9ff,color:#fff
    style ROOT fill:#d63031,stroke:#ff7675,color:#fff,font-weight:bold
```

### Failure 2 — Wrong Port

To make things worse: I initially had the cable plugged into the **WAN/Internet port** of the TP-Link instead of a LAN port.

> [!WARNING]
> **WAN port ≠ LAN port.** Consumer routers have 1 WAN port + 4 LAN ports. The WAN port has special behavior — in most modes it applies NAT, which creates a second layer of NAT between pfSense and your devices. Always use a **LAN port** when bridging.

```mermaid
flowchart LR
    subgraph TP ["TP-Link Router"]
        direction TB
        WAN_PORT["🔴 WAN Port\n(Internet In)\n— NAT applied\n— wrong port"]
        LAN1["🟢 LAN1\n(correct port)"]
        LAN2["⚫ LAN2"]
        LAN3["⚫ LAN3"]
        LAN4["⚫ LAN4"]
    end

    USB["USB Ethernet\n(from pfSense vmbr1)"]
    USB -->|"❌ was plugged here"| WAN_PORT
    USB -->|"✅ should go here"| LAN1

    style WAN_PORT fill:#d63031,stroke:#ff7675,color:#fff
    style LAN1 fill:#00b894,stroke:#55efc4,color:#fff
    style USB fill:#6c5ce7,stroke:#a29bfe,color:#fff
```

---

## ✅ The Fix — Manual AP Configuration

I factory reset the TP-Link, ignored the "AP Mode" button entirely, and configured it manually:

```mermaid
flowchart TD
    RESET["🔄 Factory Reset\nStart clean"] --> STEP1

    STEP1["1️⃣ Keep it in Router Mode\n(don't touch the AP Mode toggle)"]
    STEP1 --> STEP2

    STEP2["2️⃣ Disable DHCP Server\nin TP-Link settings\n→ Devices now ask pfSense for IP"]
    STEP2 --> STEP3

    STEP3["3️⃣ Change LAN IP to 192.168.10.2\n(inside the pfSense subnet)\n→ Can still reach admin panel"]
    STEP3 --> STEP4

    STEP4["4️⃣ Move cable from\nWAN port → LAN1 port\n→ No double NAT"]
    STEP4 --> RESULT

    RESULT["✅ IT WORKS
    —
    WiFi clients get 192.168.10.x from pfSense
    All traffic routes through pfSense firewall
    TP-Link admin accessible at 192.168.10.2"]

    style RESET fill:#636e72,stroke:#b2bec3,color:#fff
    style RESULT fill:#00b894,stroke:#55efc4,color:#fff,font-weight:bold
    style STEP1 fill:#0984e3,stroke:#74b9ff,color:#fff
    style STEP2 fill:#0984e3,stroke:#74b9ff,color:#fff
    style STEP3 fill:#0984e3,stroke:#74b9ff,color:#fff
    style STEP4 fill:#0984e3,stroke:#74b9ff,color:#fff
```

### Manual AP Mode Checklist

| Setting | Value | Why |
|:---|:---|:---|
| Mode | Router Mode (not AP) | The built-in AP Mode is buggy |
| DHCP Server | **Disabled** | pfSense handles DHCP |
| TP-Link LAN IP | `192.168.10.2` | Reachable from pfSense subnet |
| Cable port | **LAN1** (not WAN) | Avoids double NAT |

> [!TIP]
> **Lesson Learned:** Built-in "AP Mode" toggles on consumer routers are unreliable. Manual configuration — **disable DHCP + use a LAN port** — gives you exact control and actually works.
