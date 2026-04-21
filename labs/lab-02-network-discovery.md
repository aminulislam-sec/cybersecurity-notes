# Network Lab 02 – Network Discovery with Nmap

## Objective

To scan the local subnet and discover all active hosts, then interpret the results
to identify each device and understand what the scan output actually reveals.

---

## Environment

- Scanner machine: Kali Linux (Dell Latitude 7410, VMware)
- Known devices on the network: Linux Mint laptop, Windows 11 host, mobile hotspot gateway
- Tool used: Nmap 7.95
- Scan type: Ping scan (-sn flag), no port scanning

---

## Command Used

```bash
nmap -sn 10.218.101.0/24
```

This scans all 256 possible addresses in the subnet and reports which ones
responded. The -sn flag tells Nmap to check only whether hosts are alive,
without probing any ports.

---

## Raw Output

Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-21 09:38 EDT
Nmap scan report for 10.218.101.76
Host is up (0.017s latency).
MAC Address: F8:28:19:15:BD:41 (Liteon Technology)
Nmap scan report for 10.218.101.121
Host is up (0.00039s latency).
MAC Address: A4:6B:B6:24:6E:22 (Intel Corporate)
Nmap scan report for 10.218.101.182
Host is up (0.0075s latency).
MAC Address: 4A:A3:8F:98:56:6B (Unknown)
Nmap scan report for 10.218.101.36
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 4.25 seconds

---

## Interpreting the Results

Four live hosts were found. Here is what each one is and how I identified it.

**10.218.101.36 — Kali Linux (the scanner itself)**

No MAC address is shown for this host. This is expected behaviour. When Nmap
runs on a machine and that machine is part of the scanned range, it detects
itself as alive but does not display its own MAC address the way it would for
a remote device. I already knew this was Kali's IP from Lab 01.

**10.218.101.76 — Linux Mint laptop**

MAC vendor shows as Liteon Technology. Liteon is a manufacturer commonly
found in laptop wireless cards. This matched my Linux Mint machine and
confirmed the identification.

**10.218.101.121 — Windows 11 host**

MAC vendor shows as Intel Corporate. Intel network adapters are extremely
common in Windows laptops and desktops. This matched the Windows 11 host
machine.

**10.218.101.182 — Default gateway (mobile hotspot)**

MAC vendor shows as Unknown. This is normal. Routers and mobile hotspots
often do not expose a recognisable vendor name, or they use randomised MAC
addresses for privacy. I had already confirmed this IP as the gateway from
Lab 01, so the identification was straightforward.

---

## What the Latency Numbers Tell You

Nmap records how long each host took to respond. These numbers are small but
meaningful.

- 10.218.101.121 (Windows): 0.00039 seconds — extremely fast. This machine
  is the physical host that Kali is running inside as a virtual machine.
  Traffic between them barely has to travel at all.

- 10.218.101.182 (gateway): 0.0075 seconds — normal for a local router or
  hotspot.

- 10.218.101.76 (Linux Mint): 0.017 seconds — slightly higher because this
  is a completely separate physical device communicating over Wi-Fi.

This shows that latency is not just noise. It reflects physical reality:
virtual machines on the same host respond faster than separate physical
devices, and wired connections typically respond faster than wireless ones.

---

## What If a Device Did Not Appear in the Scan?

This is worth thinking about carefully, because a missing host does not always
mean the device is off.

**The device is powered off or disconnected.** The most obvious reason. If the
machine is not on the network, it cannot respond.

**A firewall is blocking ICMP packets.** Nmap's ping scan works by sending
ICMP echo requests — the same protocol used by the `ping` command. Some
operating systems and firewalls are configured to silently drop these packets.
Windows in particular does this by default on some network profiles. The device
is alive and reachable, but it does not reply to ping, so Nmap reports nothing.

**The device is on a different subnet.** If a device is connected to a
different network — a different Wi-Fi, a VPN, or a different hotspot — it will
not appear in a scan of this range, even if it is physically nearby.

**The IP address changed.** Because DHCP assigns addresses dynamically, a
device might have a different IP than expected. It could be alive in the subnet
but at an address you did not anticipate.

**The scan ran too fast.** On some networks, especially wireless ones,
devices take a moment to respond. A very fast scan can occasionally miss a
slow-responding host. Running the scan again or using a slower timing option
can help.

**The device is intentionally hiding.** In a real network, some hosts are
configured to be as invisible as possible. This is relevant in security work
because it means a clean scan result does not guarantee there are no other
devices present.

The practical lesson here is that a missing result requires investigation, not
just assumption. A missing device and a silent device look identical to the
scanner.

---

## Skills Practised

- Running a subnet-wide ping scan with Nmap
- Reading and interpreting Nmap output
- Using MAC address vendor information to identify devices
- Reading latency data as a clue about physical network topology
- Thinking critically about what scan results do and do not confirm

---

## Connection to Previous Lab

Lab 01 established the IP addresses and subnet manually by reading each
machine's configuration. Lab 02 confirmed the same information from the
outside, using only the network itself. Both approaches are necessary in
practice: knowing your own configuration and knowing what others can see
about you are two different things.
