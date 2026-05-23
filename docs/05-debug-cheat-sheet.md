# 05. Debug Cheat Sheet

> Because debugging is the learning. Here are all the commands I used across this project to figure out what was broken, what was active, and where traffic was flowing.

---

## Proxmox / Linux Shell Commands

| Command | What It Shows | When I Used It |
| :--- | :--- | :--- |
| `ip a` | Lists all network interfaces and their current IPs. | To check if the USB ethernet adapter was detected and if it was in `state UP` or `state DOWN`. |
| `ip link set <iface> up` | Forces a network interface to come online. | When the USB adapter showed `state DOWN`. (Note: This doesn't fix broken hardware, as I learned the hard way). |
| `ss -tulnp \| grep <port>` | Checks if a specific port is actively listening. | Used `ss -tulnp \| grep 8006` to prove that the Proxmox UI was running, meaning the port forwarding failure was the router's fault, not Proxmox's. |
| `curl ifconfig.me` | Returns your public-facing IPv4 address. | Used to determine if I was behind CGNAT. If this matches the router's WAN IP, you have a real public IP. |
| `lsusb` | Lists all connected USB devices. | Used to prove that the Proxmox host OS was recognizing the RTL8153 USB adapter, even when no link was established. |
| `ethtool <iface>` | Shows physical link status and speed. | To see if the ethernet cable was physically negotiating a connection with the switch. |
| `dmesg \| grep r8152` | Filters kernel messages for the Realtek driver. | To check if the driver was crashing or failing to initialize the USB adapter. |

## Network Connectivity Testing

| Command | What It Does | Why It Matters |
| :--- | :--- | :--- |
| `ping 10.27.27.1` | Pings the pfSense LAN gateway. | Proves that the Wi-Fi client is successfully bridged to the pfSense virtual network. |
| `ping 8.8.8.8` | Pings Google's DNS servers. | Proves that outbound internet routing and NAT are working through pfSense. |
| `ping google.com` | Pings a hostname. | Proves that DNS resolution (via pfSense DNS resolver) is functioning correctly. |

---

## Key Lessons

1. **Test from the outside:** If port forwarding "doesn't work," ensure you are testing from **mobile data** (outside the network). Many routers do not support NAT loopback, so testing from the same Wi-Fi will result in a timeout even if the rule is perfect.
2. **Hardware over software:** If an interface has no LED lights blinking upon cable insertion, stop typing `ip link` commands. It's likely dead hardware.
3. **LAN-to-LAN bridging:** When using a router as a dumb access point, always connect the incoming cable to a **LAN port**, never the WAN port. Disable its DHCP server so the main firewall (pfSense) handles IP assignment.
