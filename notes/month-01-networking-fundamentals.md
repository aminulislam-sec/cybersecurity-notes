# Month 1: Networking Fundamentals

**Started:** April 2026
**Completed:** May 2026
**Machines used:** Kali Linux (Dell Latitude 7410, VMware) + Linux Mint (Asus laptop)
**Labs completed:** 10

---

## 1. LAN vs WAN — What I Actually Understood

Before this month I had heard the words LAN and WAN but I could not have
told you what they meant in a way that connected to anything real. Now I
can, and I want to write it in a way that a ten-year-old could follow.

A LAN is a group of devices that can talk to each other directly, without
leaving the building. My Kali machine, my Linux Mint laptop, and the mobile
hotspot — all three of them formed a LAN during these labs. They were all
on the same subnet (10.244.116.x), they shared the same gateway, and they
could ping each other without any packet leaving the local network. That
is a LAN. You could also think of it as the conversations inside one
classroom — everyone can speak to everyone directly, no messenger needed.

A WAN is what happens when you need to reach something outside that room.
When I ran `traceroute google.com` in Lab 07, my packets left my hotspot,
passed through my ISP's routers, crossed multiple cities, and eventually
landed on a Google server. Every router along the way was part of the WAN.
The internet itself is the biggest WAN in existence.

The router sits in the middle of these two worlds. It has a private IP
facing my LAN (10.244.116.203 in my case) and a public IP facing the
internet. Everything that leaves the LAN gets translated — my private
address is swapped for the public one before the packet goes out. This is
NAT, and I saw it clearly in Lab 10 when both my machines, despite having
different private IPs, showed the same public IP (123.253.135.143) from
`curl ifconfig.me`.

The security angle I found most interesting: once an attacker gets inside
a LAN — through a phishing email, a compromised device, or physical access
to the network — they can communicate with every other device directly.
There is no outer wall protecting devices from each other inside the same
subnet. That is why what is running on each machine matters so much, which
is exactly what Labs 03 and 08 were about.

---

## 2. My Network Details from the Labs

These values are from the final bridged-mode session in Lab 10. They change
between sessions because DHCP assigns them dynamically, but the structure
stays the same.

```
Machine:            Kali Linux              Linux Mint
Private IP:         10.244.116.36           10.244.116.76
Subnet mask:        /24 (255.255.255.0)     /24 (255.255.255.0)
Default gateway:    10.244.116.203          10.244.116.203
DNS server:         10.244.116.203          127.0.0.53 (local stub)
Public IP:          123.253.135.143         123.253.135.143
Interface:          eth0                    wlp3s0
```

The DNS difference between the two machines is something I only understood
properly in Lab 10. Kali sends DNS queries directly to the hotspot. Mint
sends them to its own local resolver first, which caches answers and then
forwards to the hotspot if needed. Same destination, one extra step on Mint.

---

## 3. Commands I Learned and What They Actually Do

### ip a / ip addr

This shows every network interface on the machine and the IP address
assigned to each one. Before this month I would not have known what eth0
or wlp3s0 meant. Now I know that eth0 is a wired or virtual ethernet
interface and wlp3s0 is a wireless card. The /24 after the IP address is
the subnet mask written in a shorter form. I used this command to start
every single lab.

### ip route

This shows where the machine sends traffic that it does not know the
specific route for. The line starting with `default via` tells me the
gateway — the first stop for anything leaving the local network. I used
this in Lab 01, and I used it again in Lab 10 when I was diagnosing why
the two machines were on different subnets.

### nmap -sn (ping sweep)

This sends a ping to every address in a subnet and reports which ones
replied. In Lab 02 I used it to confirm that all four devices I expected
to find — Kali, Mint, Windows, and the gateway — were visible on the
network. It also returned MAC addresses, which helped identify which
device was which without needing to check each one individually.

### nmap (port scan)

This knocks on every port of a target machine and reports which ones
answered. In Lab 03 I discovered that Linux Mint showed no open ports at
all from the outside, even though it was clearly running services. That
result — a machine that is alive but completely silent to port scanning —
taught me more about firewalls than any explanation would have.

### ping

Sends a small packet to a destination and measures how long the reply
takes. I used it in Lab 07 to compare latency from Kali and Mint to the
gateway, to each other, and to Google. The numbers told a real story —
Kali was faster to the gateway because it is a virtual machine on the
same physical hardware as Windows, while Mint was a separate physical
device communicating over wireless.

### traceroute / tracepath

Shows every router a packet passes through on the way to a destination.
In Lab 07 I traced the path from both machines to Google and counted
21 hops. I could see my ISP's internal routers (10.32.x.x and 10.99.x.x),
then the ISP backbone (123.49.x.x), then Google's own infrastructure
(142.251.x.x) from hop 13 onward. The `* * *` lines where routers did
not respond taught me that silence is not the same as absence.

### ss -tulnp / netstat -tulnp

Lists every service currently listening for connections on the machine,
with the port number, protocol, bind address, and process name. In Lab 08
I found that Kali had PostgreSQL listening on 127.0.0.1 (localhost only,
safe) and SSH on 0.0.0.0 (all interfaces, reachable from the network).
Linux Mint had more services but almost all were localhost-only. The one
exception was avahi-daemon, which was broadcasting on the network.

### nmcli device show

Pulls the complete network profile into one readable output — IP address,
subnet, gateway, DNS server, interface type, and connection name. I used
this in Lab 10. It is faster than running ip a, ip route, and cat
/etc/resolv.conf separately, and it shows everything in one place.

### curl ifconfig.me

Asks an external server what IP address it sees the request coming from.
The answer is always the public IP — the address the internet sees, not
the private address inside the LAN. In Lab 10 this confirmed that both
machines, despite having different private IPs, were exiting to the
internet through the same address: 123.253.135.143.

### Wireshark (GUI)

Captures every packet passing through an interface and lets me open each
one and read the contents. In Lab 09 I watched a single visit to google.com
from start to finish — DNS query, TCP handshake, TLS negotiation, data
transfer. The most unexpected thing I found was that even in an encrypted
HTTPS connection, the domain name (www.google.com) is visible in plain text
in the Client Hello packet before encryption is fully established.

---

## 4. The OSI Model — Theory I Can Now Connect to Real Things

I studied the OSI model as theory this month. Seven layers, each with a
job. I used the mnemonic "All People Seem To Need Data Processing" to
remember the order from top to bottom: Application, Presentation, Session,
Transport, Network, Data Link, Physical.

What made it stick was connecting each layer to things I had actually seen
in the labs.

Layer 1 (Physical) — the cables and radio waves. My Linux Mint connects
wirelessly. My Kali VM connects through a virtual ethernet adapter inside
VMware. Both are Layer 1 implementations.

Layer 2 (Data Link) — MAC addresses. In Lab 02 Nmap returned MAC addresses
alongside IP addresses. In Lab 09 Wireshark showed MAC addresses in the
Ethernet tab of the Endpoints statistics. The OUI prefix of the Kali VM's
MAC (00:50:56) identified it as a VMware device without me needing to
know the IP.

Layer 3 (Network) — IP addresses and routing. Everything in Labs 01, 02,
07, and 10 lived here. This is where ip route and the gateway live.

Layer 4 (Transport) — TCP and UDP. The three-way handshake I watched in
Lab 09 (SYN, SYN-ACK, ACK) is Layer 4. The port numbers in ss and Nmap
output are Layer 4. The difference between TCP (reliable, ordered) and
UDP (fast, no guarantees) matters for security because UDP's connectionless
nature is what makes amplification attacks possible.

Layer 7 (Application) — the HTTP request Firefox sent in Lab 09, the DNS
query my machine sent before loading Google, the SSH connection Kali
accepted on port 22. These are all application layer conversations.

The most useful insight from studying the OSI model is that security
problems can live at any layer. A SYN flood attack breaks Layer 4. An
unencrypted HTTP login form is a Layer 7 problem. ARP spoofing attacks
Layer 2. Knowing which layer a problem lives on helps figure out where
to look for the solution.

---

## 5. TCP and UDP — and Why the Difference Matters for Security

TCP makes a connection before sending anything. The three-way handshake —
SYN, SYN-ACK, ACK — confirms both sides are ready. If a packet gets lost,
TCP resends it. This reliability has a cost: speed. But for things like
SSH sessions, file transfers, and web pages, accuracy matters more than
raw speed.

UDP just sends and hopes. No handshake, no confirmation, no retransmission.
I saw UDP in Lab 09 when Google used QUIC — a UDP-based protocol — for
part of the connection. Google chose UDP there because loading a web page
fast is more important than guaranteeing every packet arrives in perfect
order.

The security angle that I want to remember: TCP's handshake is also its
weakness. A SYN flood attack sends thousands of SYN packets but never
sends the final ACK. The server holds open thousands of half-finished
connections waiting for replies that never come. Eventually it runs out
of memory. No exploit needed — just an abuse of normal TCP behaviour.

---

## 6. Ports I Now Recognise on Sight

Before this month, port numbers meant nothing to me. Now I look at a port
number and I have a context for it.

Port 22 is SSH — remote login. I found it open on Kali in Lab 03 and Lab 08.
On a machine with a public IP and weak passwords, this is a brute-force
target that gets hit constantly.

Port 80 is HTTP — unencrypted web traffic. Everything travels in plain
text. In Lab 09 I saw Firefox's captive portal check go out over HTTP
and I could read every word of the request and response in Wireshark.

Port 443 is HTTPS — encrypted web traffic. All 91.6% of the packets in
my Google visit were TLS-encrypted traffic on this port. Encrypted, but
the domain name is still visible in the Client Hello.

Port 5432 is PostgreSQL. I found it running on Kali in Labs 03 and 08,
correctly bound to 127.0.0.1. The nc test from Mint confirmed it was
genuinely unreachable from the network — not just hidden, but actually
inaccessible.

Port 5357 appeared on Windows in Lab 03. That was Windows' Web Services
Discovery service, which helps local devices find shared printers and
folders. I had not opened it deliberately — Windows opens it automatically.

---

## 7. What Confused Me — and How I Worked Through It

**The VMware network mode switch.** In Lab 10 I started a session and the
two machines were on different subnets. I did not immediately know why.
I had to think through what had changed, and eventually found that VMware
had switched from bridged to NAT mode at some point between sessions. Once
I understood the difference — bridged puts the VM on the real network, NAT
hides it behind Windows — the fix was obvious. But getting to that
understanding took time I had not planned for. I now check the network
mode at the start of any session where cross-machine work is planned.

**The filtered port result on Linux Mint.** In Lab 03 I ran five different
Nmap scans on Mint and every single one came back with all ports filtered.
I spent a while wondering if Nmap was broken or if I was doing something
wrong. The answer was that Mint's firewall was dropping every TCP probe
silently, and that is exactly the correct behaviour for a well-configured
machine. A machine that shows nothing is not empty — it is protected. I
needed to see that result five times before I believed it.

**The /24 subnet notation.** Early in Month 1 I saw /24 everywhere and
treated it as a detail I could ignore. Eventually I stopped ignoring it
and looked it up properly. /24 means the first 24 bits of the IP address
identify the network, leaving 8 bits for individual devices. That gives
256 possible addresses, with 254 usable. Knowing this made the subnet
comparisons in Labs 01 through 10 much more readable.

**Tracepath stopping at hop 13.** In Lab 07 both machines' tracepath
output stopped responding after reaching Google's network, reporting "Too
many hops." I thought the scan had failed. Then I ran traceroute and it
continued all the way to hop 21. The difference is that tracepath and
traceroute probe differently, and Google's internal routers respond to
one but not the other. The destination was never unreachable — the tool
had just hit a wall that the other tool could see past.

---

## 8. Security Observations from This Month

The finding that stayed with me most was from Lab 09. I had assumed that
HTTPS meant private. It does — the content is encrypted. But the domain
name appears in plain text in the Client Hello packet before encryption
is established. My ISP can see every domain I visit even when I use HTTPS.
A corporate firewall can see the same. That is not a bug. It is how TLS
currently works, and I had not known it before I watched a real connection
in Wireshark.

The second observation is about default settings. Port 5357 was open on
Windows because Windows opened it automatically — I had never made a
conscious decision about it. Avahi-daemon was broadcasting on Linux Mint
for the same reason. SSH was open on Kali because it is installed by
default. None of these are malicious. But every one of them is an attack
surface that exists without the user choosing it. The lesson I took from
Labs 03 and 08 is that a machine's default state is not its secure state.
Security requires active decisions, not just the absence of bad ones.

---

## 9. Questions I Am Carrying into Month 2

How does SSH key authentication work, and how does it compare to password
authentication in terms of what an attacker would need to break in?

I saw QUIC in Wireshark in Lab 09. It uses UDP but still provides
encryption. How does that work when UDP has no handshake?

Avahi-daemon broadcasts the machine's hostname and services on the local
network. Exactly what information does it expose, and would disabling it
break anything I actually use?

In Lab 08 I found PostgreSQL running on Kali. I did not install it. What
else is running on my machines that I did not deliberately put there, and
how do I audit for that systematically?

What is the command line in Linux actually doing differently from a GUI,
and why do security practitioners prefer it? I have been using terminals
for a month but I have not yet asked that question properly.

---

## Closing Note

The thing I understood least at the start of Month 1 was that reading
output is a skill. I could run commands from the first day. But looking
at the result of ip a or an Nmap scan and knowing what it meant — knowing
which number was the gateway, which part was the subnet, what filtered
versus closed actually signified — that took the whole month to build.

What surprised me most was how much the tools confirm each other. The IP
address from ip a matched what Nmap found in the discovery scan, which
matched what Wireshark showed as the source address in captured packets,
which matched what nmcli reported. Every tool was looking at the same
network from a different angle. Once I understood that, I stopped treating
each lab as a separate exercise and started seeing them as one connected
picture.

Month 2 is Linux CLI. I am starting it with a clearer sense of what I am
actually trying to learn — not commands, but the thinking behind them.
