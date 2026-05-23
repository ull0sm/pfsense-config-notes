# 🔧 pfSense Installation Guide — Step by Step

> Exactly how pfSense was installed on Proxmox in this project. Reproducible and battle-tested.

---

## Prerequisites

- [x] Proxmox VE installed and accessible
- [x] Internet working via `vmbr0`
- [x] Two network interfaces (built-in ethernet + USB ethernet adapter)
- [x] Backup remote access via Tailscale
- [x] Physical access to the machine (recommended during setup)

---

## Step 1: Download pfSense ISO

1. Go to [pfSense CE Downloads](https://www.pfsense.org/download/)
2. Select:
   - Architecture: **AMD64 (64-bit)**
   - Installer: **USB Memstick Installer**  
   - Console: **VGA**
3. Download and extract the `.iso` file

---

## Step 2: Upload ISO to Proxmox

In the Proxmox web UI:

```
Datacenter → local (storage) → ISO Images → Upload
```

Select the pfSense ISO and upload it.

---

## Step 3: Create the Internal LAN Bridge (vmbr1)

This is the **most important preparation step** and is completely safe.

In Proxmox:

```
Node → Network → Create → Linux Bridge
```

Configure `vmbr1`:

| Setting | Value |
|---------|-------|
| Name | `vmbr1` |
| Bridge ports | **EMPTY** (no physical port initially) |
| CIDR | **EMPTY** |
| Gateway | **EMPTY** |

Click **Create**, then **Apply Configuration**.

> ⚠️ **Why this is safe:** Creating an empty bridge with no IP, no gateway, and no ports cannot break your existing internet or Proxmox access. It's just creating a virtual switch that doesn't connect to anything yet.

### After Adding USB Ethernet

Once you have the USB ethernet adapter plugged in and detected:

```bash
# Verify USB adapter is detected
lsusb
# Should show: Realtek Semiconductor Corp. RTL8153 Gigabit Ethernet Adapter

# Find interface name
ip a
# Look for: enx00e04c314578 or similar (state DOWN is normal)

# Bring interface up
ip link set enx00e04c314578 up

# Verify link
ethtool enx00e04c314578
# Look for: Link detected: yes
```

Edit `vmbr1` to add the USB ethernet as bridge port:

| Setting | Value |
|---------|-------|
| Bridge ports | `enx00e04c314578` (your USB NIC name) |

Apply configuration.

### Result

| Bridge | Physical Port | Purpose |
|--------|--------------|---------|
| `vmbr0` | `nic0` (built-in) | WAN — internet uplink |
| `vmbr1` | USB ethernet | LAN — internal network |

---

## Step 4: Create pfSense VM

In Proxmox: **Create VM**

### General
| Setting | Value |
|---------|-------|
| VM ID | Pick any (e.g., 101) |
| Name | `pfsense` |

### OS
| Setting | Value |
|---------|-------|
| ISO | Select the uploaded pfSense ISO |
| Type | Other |

### System
| Setting | Value |
|---------|-------|
| BIOS | **OVMF (UEFI)** |
| Machine | **q35** |
| EFI Storage | local-lvm |

### Disks
| Setting | Value |
|---------|-------|
| Size | **20 GB** |
| Storage | local-lvm |

### CPU
| Setting | Value |
|---------|-------|
| Cores | **2** |

### Memory
| Setting | Value |
|---------|-------|
| RAM | **4096 MB** (4 GB) |

### Network (NIC 1 — WAN)
| Setting | Value |
|---------|-------|
| Bridge | **vmbr0** |
| Model | **VirtIO (paravirtualized)** |

### Add Second NIC (NIC 2 — LAN)

After creation, go to VM → Hardware → Add → Network Device:

| Setting | Value |
|---------|-------|
| Bridge | **vmbr1** |
| Model | **VirtIO (paravirtualized)** |

---

## Step 5: Install pfSense

1. **Start the VM** and open the console (noVNC)
2. Boot from the ISO
3. Accept the copyright notice
4. Select **Install pfSense**
5. Choose keymap (default is fine)
6. Partitioning: **Auto (ZFS)** or **Auto (UFS)** — either works
7. Wait for installation to complete
8. **Remove the ISO** from VM hardware (CD/DVD drive)
9. **Reboot** the VM

---

## Step 6: Assign Interfaces

On first boot, pfSense will ask to assign interfaces.

**Do NOT set up VLANs** (answer `n`)

Then assign:

| Prompt | Interface | Bridge |
|--------|-----------|--------|
| WAN | `vtnet0` | vmbr0 |
| LAN | `vtnet1` | vmbr1 |

> **Tip:** If unsure which is which, check MAC addresses in Proxmox VM hardware and match them.

---

## Step 7: Configure LAN IP

From the pfSense console menu, select option **2** (Set interface IP address).

Choose the **LAN** interface and set:

| Setting | Value |
|---------|-------|
| IPv4 address | `10.27.27.1` |
| Subnet | `24` |
| IPv6 | Skip (press Enter) |
| Enable DHCP | **Yes** |
| DHCP range start | `10.27.27.100` |
| DHCP range end | `10.27.27.200` |
| Revert to HTTP | **No** (keep HTTPS) |

---

## Step 8: Set Up TP-Link Router as Access Point

The TP-Link router becomes a **dumb WiFi access point** — no routing, no DHCP.

### Manual Configuration (More Reliable Than AP Mode)

1. Connect to TP-Link directly (via WiFi or cable)
2. Access admin page (usually `192.168.0.1` or `tplinkwifi.net`)
3. **Disable DHCP** server
4. **Change LAN IP** to `10.27.27.2` (inside pfSense subnet)
5. Keep WiFi settings as-is

### Physical Connection

```
pfSense LAN (USB Ethernet)
        │
   Ethernet Cable
        │
TP-Link LAN Port (NOT the WAN/Internet port!)
```

> ⚠️ **Common mistake:** Plugging into the WAN/Internet port instead of a LAN port. The cable MUST go into LAN1-4, NOT the WAN port.

> ⚠️ **TP-Link AP mode issues:** The built-in "AP Mode" on some TP-Link routers disables LAN ports or behaves strangely. Manual configuration is usually more reliable.

---

## Step 9: Verify Connectivity

Connect a device to TP-Link WiFi and check:

| Check | Expected Value |
|-------|---------------|
| IP Address | `10.27.27.x` |
| Gateway | `10.27.27.1` |
| DNS | `10.27.27.1` |
| Internet | Working ✅ |

If the device gets a `10.27.27.x` address, **pfSense DHCP is working**.

---

## Step 10: Complete pfSense Setup Wizard

Open browser and navigate to:

```
https://10.27.27.1
```

Login:
- Username: `admin`
- Password: `pfsense`

### Wizard Settings

| Setting | Value |
|---------|-------|
| Hostname | `pfsense` |
| Domain | `home.arpa` |
| Primary DNS | `1.1.1.1` |
| Secondary DNS | `8.8.8.8` |
| Timezone | `Asia/Kolkata` |
| WAN Type | **DHCP** (gets IP from Airtel router) |
| LAN IP | `10.27.27.1/24` |
| Admin Password | **Change immediately!** |

---

## Step 11: Post-Installation Tuning

### Update pfSense

```
System → Update
```

Install all available updates.

### Disable Hardware Checksum Offloading

```
System → Advanced → Networking
```

Check: **☑ Disable hardware checksum offload**

> ⚠️ **CRITICAL for VMs.** Without this, you'll get random packet corruption and connectivity issues. This is a known issue with VirtIO NICs.

### Verify DNS Resolver

```
Services → DNS Resolver
```

Ensure **"Enable DNS Resolver"** is checked.

---

## Step 12: Take a Proxmox Snapshot

**This is your safety net.** If anything breaks later, roll back to this point.

```
Proxmox → pfSense VM → Snapshots → Take Snapshot
Name: "working-network"
```

---

## Troubleshooting Quick Reference

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| TP-Link shows all LAN ports "unplugged" | Bad cable, wrong port, or dead adapter | Try different cable/port, check USB adapter LEDs |
| USB NIC shows `Active = No` in Proxmox | Interface is DOWN | `ip link set <interface> up` |
| Devices get `192.168.x.x` instead of `10.27.27.x` | TP-Link DHCP still running | Disable DHCP on TP-Link |
| Devices get `169.254.x.x` (APIPA) | pfSense DHCP not reaching devices | Check physical cable path |
| Can't reach pfSense web UI | Wrong IP or interface down | Try from console: option 7 → ping 10.27.27.1 |
| Internet works but DNS fails | DNS resolver not running | Check Services → DNS Resolver |
| No internet after pfSense install | WAN interface misconfigured | Check pfSense console → option 2 → verify WAN has IP |

---

## Safety Reminders

> 🛑 **Do NOT touch `vmbr0` IP or gateway** during setup. That's your lifeline to the internet and Proxmox.

> 🛑 **Keep Proxmox management on `vmbr0`** initially. Don't move it behind pfSense until everything is stable.

> 🛑 **Be physically near the machine** during NIC reassignment and DHCP changes. One wrong move can lock you out remotely.

> ✅ **Tailscale remains working** regardless of pfSense changes — it's your emergency backup access.
