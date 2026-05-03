# Network Lab 01 – Basic IP Configuration and Connectivity

## Objective

To identify IP configuration details across three machines, confirm they are on the same network, and verify that they can communicate with each other.

---

## Environment

- Host machine: Windows 11 (Dell Latitude 7410)
- Virtual machine: Kali Linux, running inside Windows 11 via VMware
- External machine: Linux Mint on a separate laptop (Asus)

All three machines were connected to the same network during this lab.

---

## IP Configuration

**Kali Linux**

Interface: eth0
IP address: 10.252.78.36
Subnet mask: 255.255.255.0 (/24)
Default gateway: 10.252.78.54

Command used: `ip a`

**Linux Mint**

Interface: wlp3s0 (wireless)
IP address: 10.252.78.76
Subnet mask: 255.255.255.0 (/24)
Default gateway: 10.252.78.54

Command used: `ip a`

**Windows 11**

Interface: Wi-Fi
IP address: 10.252.78.121
Subnet mask: 255.255.255.0 (/24)
Default gateway: 10.252.78.54

Command used: `ipconfig`

---

## Observations

All three machines are on the same subnet: 10.252.78.0/24. Each has a unique IP address, but they share the same subnet mask and the same default gateway. This means traffic between them does not need to pass through a router — they can communicate directly at the local network level.

The IP addresses were assigned automatically by DHCP, not set manually. This is the default behavior on most networks. The addresses will likely change the next time the machines reconnect.

Interface names are different across operating systems. Kali uses eth0 for a wired connection, Linux Mint uses wlp3s0 for wireless, and Windows uses a generic "Wi-Fi" label. These are just naming conventions — the underlying function is the same.

---

## Core Concept: Same Subnet, Direct Communication

When two devices share the same network address and subnet mask, they can reach each other directly without routing. In a /24 network, this means the first three octets of the IP address must match.

In this lab: all three machines are in the 10.252.78.x range with a /24 mask, so direct communication is expected to work.

If the subnet were different — for example, one machine on 10.252.78.x and another on 192.168.1.x — they would not be able to communicate unless a router was configured to connect them.

---

## Verification Steps

Before starting any lab session, it is good practice to run these checks:

1. Confirm your IP address and active interface.
   - Linux: `ip a` or `ifconfig`
   - Windows: `ipconfig`

2. Check the routing table and confirm the default gateway.
   - Linux: `ip route`
   - Windows: `route print` or `ipconfig`

3. Verify that the first three octets of the IP address match across all machines (for a /24 network).

4. Test reachability using ping.
   - `ping <target-ip>`
For example, 
# Ping your Kali Linux 
ping 10.252.78.36

# Ping your Windows host
ping 10.252.78.121

# Ping your Linux Mint
ping 10.252.78.76

# Test internet
ping 8.8.8.8

   - A successful reply confirms the machines can see each other.

---

## Known Issues to Watch For

**VPN interference.** If a VPN like ProtonVPN is active, it changes the routing table and can block or redirect local traffic. Always disconnect from VPN before running local network labs.

**Changing IP addresses.** Because DHCP assigns addresses dynamically, the IP can change between sessions — after reconnecting to Wi-Fi, switching to hotspot, or restarting. Always re-check with `ip a` or `ipconfig` at the start of each session.

**Ping blocked by firewall.** Some systems block ICMP packets (which ping uses). A failed ping does not always mean the machines are unreachable — it may just mean ICMP is filtered. Other tools like `nmap` can be used to confirm reachability in those cases.

**Different network sources.** If one machine is on Wi-Fi and another is on mobile hotspot, they will be on different subnets and will not be able to communicate, even if they appear to have similar-looking IP addresses.

---

## What I Practised

- Running `ip a`, `ip route`, and `ipconfig` to read network configuration
- Identifying the interface name, IP address, subnet mask, and gateway on each machine
- Understanding what a /24 subnet means in practice
- Confirming that matching subnet = potential for direct communication
- Thinking about what can go wrong before blaming the tools

---

## Why This Matters

Knowing how to read IP configuration and verify connectivity is a prerequisite for everything that comes later — scanning, enumeration, service discovery, and exploitation all depend on first knowing whether the machines involved can actually reach each other. This lab established that baseline.
