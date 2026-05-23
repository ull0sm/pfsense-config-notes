# 🔥 Troubleshooting War Stories

> Every problem encountered and how it was solved. Because debugging IS the learning.

---

## War Story #1: "Port Forwarding Doesn't Work"

### Symptoms
- Port forward set up on Airtel router: 8006 → 192.168.1.240:8006
- Tested from mobile data → timeout
- Proxmox was definitely listening

### Investigation

```bash
# Confirmed service is running
root@pve:~# ss -tulnp | grep 8006
tcp   LISTEN 0   4096   *:8006   *:*   users:(("pveproxy worker"...))

# Confirmed firewall is OFF
# Proxmox UI: Datacenter → Firewall → disabled
```

### Root Cause
Multiple factors:
1. ISP may filter port 8006
2. Tested from same WiFi initially (NAT loopback issue)
3. Port 443 reserved by Airtel firmware

### Fix
- Always test from **mobile data**, never same WiFi
- Use non-standard external ports (4443 instead of 8006)
- Ultimately: **replaced ISP routing with pfSense**

### Lesson
> Port forwarding failures are rarely just "one thing." Check: ISP filtering → router rules → firewall → service → testing method.

---

## War Story #2: "Rule Conflict with Remote MGMT Setting"

### Symptom
Trying to forward WAN 443 → Ubuntu 443 on Airtel router gave:

> **"Rule conflict with Remote MGMT setting"**

### Root Cause
Airtel router firmware **hard-reserves WAN port 443** for its own HTTPS remote management interface. This cannot be disabled, changed, or overridden.

### What I Tried
| Attempt | Result |
|---------|--------|
| Different port forwarding fields | Same error |
| Hidden admin URLs (`/management.asp`, etc.) | 404 |
| Telnet/SSH to router | No access |
| Firmware update | ISP-locked |

### Fix
**There was no fix within the ISP router.** This was the catalyst for the entire pfSense project.

### Lesson
> ISP routers are not designed for self-hosting. They serve consumers. If you need real control, you need your own firewall.

---

## War Story #3: "The Dead USB Adapter"

### Symptoms
- USB-to-Ethernet adapter plugged into Proxmox
- `lsusb` detected it as RTL8153
- `ip a` showed interface but `state DOWN`
- Even after `ip link set <interface> up`, no link established
- TP-Link showed all LAN ports "unplugged"

### Investigation

```bash
# Detected by USB subsystem
root@pve:~# lsusb
Bus 004 Device 002: ID 0bda:8153 Realtek Semiconductor Corp. RTL8153

# Interface exists but DOWN
root@pve:~# ip a
3: enx00e04c314578: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN

# Tried bringing up
ip link set enx00e04c314578 up
# No change - no link LED, no TP-Link detection

# Checked driver
dmesg | grep r8152
# Driver loaded but no link detected
```

**TP-Link side:**
```
Ethernet Status:
Internet: unplugged
LAN1: unplugged
LAN2: unplugged
LAN3: unplugged
LAN4: unplugged
```

### Root Cause
The USB-to-Ethernet adapter was **physically defective**. Non-functional hardware. No LED activity whatsoever.

### Fix
Ordered a **new USB-to-Ethernet adapter** via quick commerce delivery. New adapter:
- LEDs lit up immediately
- `state UP` after `ip link set`
- TP-Link showed `LAN1: connected`
- pfSense LAN immediately started working

### Lesson
> Before spending hours debugging software, **check if the hardware actually works.** No LEDs = probably dead hardware.

---

## War Story #4: "TP-Link AP Mode Killed the LAN Ports"

### Symptom
After enabling TP-Link's built-in "AP Mode," all LAN ports showed as unplugged even with cables connected.

### Root Cause
Some TP-Link routers, when put in AP mode:
- Disable/reassign physical ports
- Change internal switching behavior  
- Require connection through a specific port only
- Reboot into a different network mode

### Fix
Instead of using the built-in AP mode, **manually configured** the TP-Link:

1. Reset TP-Link to factory defaults
2. Keep in **normal router mode**
3. **Disable DHCP** manually
4. Change LAN IP to `10.27.27.2`
5. Connect cable to a **LAN port** (NOT WAN/Internet port)

This was much more reliable than the vendor's "AP Mode."

### Lesson
> Built-in "AP Mode" on consumer routers is often buggy. Manual configuration (disable DHCP + change IP + LAN-to-LAN cable) is usually more reliable.

---

## War Story #5: "USB NIC Active = No in Proxmox"

### Symptom
Proxmox network configuration showed the USB NIC bridge port as `Active = No`.

### Root Cause
The USB ethernet interface was detected but in `state DOWN` by default. Linux doesn't automatically bring up interfaces unless configured to.

### Fix

```bash
# Bring interface up manually
ip link set enx00e04c314578 up

# Verify
ip a
# Should show: <BROADCAST,MULTICAST,UP,...>

# For persistence, add to /etc/network/interfaces
# or configure through Proxmox bridge settings
```

### Lesson
> USB network interfaces often default to DOWN state. Always explicitly bring them up and verify with `ip a`.

---

## War Story #6: "Wrong TP-Link Port"

### Symptom
Cable connected to TP-Link but no link established.

### Root Cause
TP-Link routers have:
```
1 WAN port + 4 LAN ports
```

The cable was accidentally plugged into the **WAN/Internet port** instead of a **LAN port**.

In AP mode (or when used behind another router), you must use **LAN-to-LAN** connections, not LAN-to-WAN.

### Fix
Moved cable from WAN port to LAN1.

### Lesson
> Always use LAN ports when connecting an access point behind another router/firewall. The WAN/Internet port is for upstream connections only.

---

## War Story #7: "NAT Loopback Testing Confusion"

### Symptom
Port forwarding appeared "broken" because accessing `https://122.172.83.46:8006` from home WiFi timed out.

### Root Cause
The Airtel router doesn't support **NAT loopback** (also called NAT hairpinning). This means you can't access your own public IP from inside the same network.

### Fix
Always test external access from:
- **Mobile data** (disconnect from WiFi first)
- A friend's network
- An online port checker website

### Lesson
> If port forwarding "doesn't work," first make sure you're testing from OUTSIDE your network. Many consumer routers don't support NAT loopback.

---

## Debug Command Cheat Sheet

| Command | What It Shows |
|---------|--------------|
| `ip a` | All network interfaces and IPs |
| `ip link set <iface> up` | Bring interface up |
| `ss -tulnp \| grep <port>` | Check if service is listening |
| `curl ifconfig.me` | Your public IP |
| `lsusb` | Connected USB devices |
| `ethtool <iface>` | Link status and speed |
| `dmesg \| grep r8152` | RTL8153 USB driver messages |
| `ping 10.27.27.1` | Test pfSense LAN connectivity |
| `ping 8.8.8.8` | Test internet connectivity |
| `ping google.com` | Test DNS resolution |
