# 03. The Final Verdict (The CGNAT Reality)

> **TL;DR:** After building a complex pfSense virtualization setup to bypass my ISP router's port 443 lock, I discovered the ultimate roadblock: Airtel FTTH put me behind a Carrier-Grade NAT (CGNAT). The direct hosting dream is officially dead.

---

## The Discovery

I thought I had a real public IP. During my initial checks, I somehow saw matching IPs, or perhaps Airtel dynamically moved me to a shared pool later. Either way, when I finally got everything working—pfSense routing, Proxmox bridges, TP-Link in true AP mode—I still couldn't access my web apps from the outside.

I dug deeper into my pfSense WAN interface (which gets its IP via DHCP from Airtel). 
**The WAN IP was in a private/shared range, completely different from what `curl ifconfig.me` showed.**

I was behind a **CGNAT**.

> [!CAUTION]
> **What is CGNAT?**
> Carrier-Grade NAT means the ISP shares a single public IP address across hundreds of customers. You don't actually own a public IP; your "WAN" IP is just a private IP on the ISP's massive internal network. 
> **Result:** Port forwarding is impossible. The ISP's main router drops incoming requests long before they even reach your house.

---

## Calling Airtel Customer Care

I contacted Airtel to see if there was a way out. They offered a solution: **A Static IP**.

The cost? **₹350/month (including GST)**.

While it's a guaranteed way to get a public IP and bypass the CGNAT, I decided against it for a few reasons:
1. **The Cost:** Self-hosting was supposed to be a cheap/free alternative to the cloud. Adding a recurring monthly fee defeats the purpose.
2. **Reliability:** Based on my recent research, getting a reliable static IP on consumer broadband in India is a mixed bag. It often breaks during node maintenance or requires constant follow-ups with local field engineers.

---

## The Indian ISP Landscape (CGNAT vs. Public IP)

Interestingly, through conversations and web searches (documented in `context.txt`), I learned that you actually have a *higher* chance of getting a real public IP with **BSNL or local tier-3 broadband operators**.

Why? Because running a massive, efficient CGNAT infrastructure requires complex setup and expensive carrier-grade equipment.
- **Airtel / Jio:** They have millions of users. They *have* to aggressively use CGNAT to save IPv4 addresses. Their infra is highly optimized for it.
- **BSNL / Local ISPs:** Many local cable operators simply hand out dynamic public IPs because they don't have the technical capability or budget to implement complex CGNAT routing. Their "low infra" setup ironically makes them better for self-hosters.

---

## The Final Verdict

My attempt to directly host web apps by port forwarding 443 on Airtel FTTH is over. 

### What did I gain?
I didn't get to host my apps directly, but the journey was incredibly valuable:
- I learned how to virtualize a firewall (pfSense) in Proxmox.
- I understand Linux networking bridges (`vmbr0`, `vmbr1`).
- I know how to debug network interfaces (`lsusb`, `dmesg`, `ss -tulnp`).
- I finally understand the difference between AP mode and routing mode on consumer routers.

### How will I host my apps now?
Since direct port forwarding is dead, the only solutions left are tunneling:
1. **Cloudflare Tunnels (cloudflared):** Creates an outbound connection from my server to Cloudflare, bypassing CGNAT entirely.
2. **Tailscale / Zerotier:** What I'm currently using. It creates a mesh VPN so I can securely access my services, though it doesn't allow random public internet users to visit my domain easily without a VPS relay.

*Sometimes you build an entire rocket ship just to find out there's a concrete roof over the launch pad. But hey, at least I know how to build a rocket ship now.* 
