# 04. The TP-Link AP Struggle

> Once pfSense was virtualized and passing traffic through the working USB adapter, I needed a way to provide Wi-Fi to my local devices. This documents the struggle of using a consumer router as an Access Point (AP).

---

## What I Wanted To Do

I wanted the TP-Link router to act purely as a dumb Wi-Fi antenna (an Access Point). It should broadcast Wi-Fi, but leave all routing, firewall rules, and DHCP assignments to the pfSense VM.

## What I Tried

I connected an ethernet cable from the Proxmox USB adapter (`vmbr1`) into the TP-Link router. 

I logged into the TP-Link settings and used the built-in software toggle to switch the device from "Router Mode" to **"Access Point Mode"**.

## What Failed

After enabling the built-in "AP Mode", the network broke entirely.

**Symptoms:**
- The TP-Link router showed all LAN ports as "unplugged" even when the cable was connected.
- Devices connecting to the Wi-Fi were not receiving an IP address from pfSense.
- The TP-Link seemed to change its internal switching behavior and completely isolated the pfSense DHCP server.

**Another mistake:** At one point, I had plugged the cable into the **WAN/Internet port** of the TP-Link.
- Consumer routers have 1 WAN port + 4 LAN ports. 
- In AP mode (or when bridging), the WAN port acts differently or gets disabled. Connecting LAN-to-WAN meant the traffic was being blocked by the TP-Link's own NAT layer.

## The Fix

I realized the vendor's built-in "AP Mode" was buggy and unpredictable. I needed to configure it manually.

1. **Factory Reset:** I reset the TP-Link to clear the buggy AP mode.
2. **Router Mode:** I kept it in standard "Router Mode."
3. **Disable DHCP:** I manually disabled the DHCP server inside the TP-Link settings. This forces all connected Wi-Fi devices to ask the upstream pfSense router for an IP address.
4. **Change LAN IP:** I changed the TP-Link's LAN IP to `10.27.27.2` (in the pfSense subnet) so I could still access its admin panel.
5. **LAN-to-LAN Connection:** I moved the physical ethernet cable from the WAN port to **LAN1**.

> [!SUCCESS]
> **Result:** It worked perfectly. By connecting LAN-to-LAN with DHCP disabled, the TP-Link router stopped trying to route traffic and simply bridged the Wi-Fi connections directly to pfSense. Devices instantly received `10.27.27.x` IPs.

> [!TIP]
> **Lesson Learned:** Built-in "AP Mode" toggles on consumer routers are often unreliable. Manual configuration (disabling DHCP + changing IP + using a LAN-to-LAN cable connection) gives you exact control over how the bridge behaves.
