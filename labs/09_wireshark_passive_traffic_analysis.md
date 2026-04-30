# Lab 09 – Watching the Network: Passive Traffic Analysis with Wireshark

## What This Lab Is About

Every lab so far looked at the network from a command line — checking IP
addresses, scanning for open ports, listing listening services. This lab
did something different. Instead of asking questions and waiting for
answers, I just watched.

Wireshark is a tool that captures every packet passing through a network
interface and lets you open each one and read what is inside. It does not
send anything, block anything, or change anything. It only observes. That
is called passive capture.

Three questions were investigated in this lab, each as a separate project.

---

## A Simple Way to Think About This

Imagine standing on a footbridge above a busy road. Cars pass underneath
carrying parcels from place to place. You cannot stop the cars. You cannot
open the parcels. But you can write down every number plate you see, note
which direction each car came from, and spot patterns — the same car
making the same trip repeatedly, some cars arriving faster than others,
some carrying unusually large loads.

Wireshark is that footbridge. The packets are the cars. Everything you
learn, you learn by watching.

---

## Environment

- Machine: Kali Linux, 192.168.161.128, inside VMware on Windows 11
- Interface captured: eth0
- Local DNS and gateway: 192.168.161.2
- Tool: Wireshark 4.4.9
- All captures performed on a private VMware virtual network

---

## Before the First Capture — Setup and Safety

### Why not run Wireshark as root

The first instinct was to run `sudo wireshark`. That works, but it is the
wrong approach. Wireshark has a large amount of code that reads and decodes
packets. If a malicious packet exploited a bug in that code while the
whole application was running as root, an attacker could take over the
machine completely.

The safer design is to let only a small background program called dumpcap
run with elevated privileges. The rest of Wireshark runs as a normal user.
This is called the principle of least privilege — give each process only
the permissions it genuinely needs, nothing more.

This was configured with two steps:

```bash
sudo usermod -aG wireshark $USER
sudo dpkg-reconfigure wireshark-common
# Selected "Yes" when asked if non-superusers can capture packets
```

Then logged out and back in. Confirmed the setup worked:

```bash
groups $USER
# Output included: wireshark
```

From this point forward, Wireshark runs with no sudo.

### A mistake worth recording

After completing the group setup, `sudo tshark` was run once out of habit.
Nothing broke — a capture command only reads packets and exits. But the
setup had just created a proper security boundary, and then it was
bypassed immediately. The lesson is not just about which command to type.
It is about understanding why the boundary exists and respecting it.

### Another small mistake

Typing `eth0` as a standalone command produced `eth0: command not found`.
Interface names like eth0 are not programs. They are labels — arguments
passed to tools like `ip a` or `tshark -i eth0`. The correct way to see
available interface names is:

```bash
ip a
```

---

## The Workflow Used for Every Project

Before this lab there was no system for using Wireshark. Without one,
the tool produces a scrolling wall of coloured rows that is hard to make
sense of. The workflow that made this lab coherent:

1. Write the question in one sentence before touching anything
2. Set a capture filter before starting — keep it narrow
3. Perform exactly one action during the capture
4. Stop the capture immediately after that action
5. Apply display filters to isolate the relevant packets
6. Use Follow > TCP Stream or Follow > UDP Stream to read full conversations
7. Use the Statistics menu for summary data
8. Write a conclusion that directly answers the opening question

---

## Project 1 — What Happens When I Visit a Website?

### The question

When google.com is typed into a browser and Enter is pressed, what happens
at the network level, in what order, and how long does each step take?

### What was done

Capture filter set to `host google.com`. Wireshark started. Google.com
typed into Firefox. Page loaded. Capture stopped. One website visit,
nothing else.

---

### Step 1 — DNS: The Name Lookup

The very first thing that happened — before a single image or word from
Google loaded — was a question: "What is the IP address for google.com?"
This is a DNS query. It travels on UDP port 53.

Two linked packets appeared: packet 169 (the question) and packet 172
(the answer). Wireshark showed the link directly — at the bottom of packet
169: `[Response In: 172]`, and at the bottom of packet 172:
`[Request In: 169]`.

**Packet 169 — the DNS query**

```
Time:              12.382929200s
Source IP:         192.168.161.128   (Kali)
Destination IP:    192.168.161.2     (local DNS / gateway)
Protocol:          DNS over UDP
Source port:       43419             (random temporary port)
Destination port:  53                (always DNS)
Transaction ID:    0xe29c            (a ticket number — response must match)
Query type:        A                 (asking for an IPv4 address)
```

**Packet 172 — the DNS response**

```
Time:              12.391857453s
Transaction ID:    0xe29c            (same ticket — confirmed match)
Flags:             No error
Answer RRs:        6                 (six IP addresses returned)
```

The six addresses Google returned:

```
74.125.24.100
74.125.24.101
74.125.24.102
74.125.24.113
74.125.24.138
74.125.24.139
```

Google returns multiple addresses on purpose. This is called load
balancing. The browser connects to whichever responds fastest. If one
server goes down, the others continue serving. This is how Google handles
billions of users without any single point of failure.

DNS resolution time: 12.391857453 − 12.382929200 = **8.9 milliseconds**.
Under 10ms means the gateway had the answer cached already. A slow or
distant DNS server typically takes 100–300ms.

**The IPv6 query that ran in parallel**

At almost the same moment, a second DNS question went out — this time for
Google's IPv6 address (query type AAAA). Modern systems ask for both IPv4
and IPv6 simultaneously and use whichever the network prefers. These were
two separate conversations with different transaction IDs, running in
parallel.

---

### Step 2 — TCP: Opening the Connection

With Google's IP address known, a connection needed to be opened. This
started with a SYN packet — the network equivalent of knocking on a door
before entering.

Display filter used: `tcp.flags.syn==1`

The first SYN packet going to Google's IP was right-clicked and Follow >
TCP Stream selected. The handshake appeared as:

```
My computer → Google:   SYN       (I want to connect)
Google → My computer:   SYN-ACK   (acknowledged, come in)
My computer → Google:   ACK       (connection confirmed)
```

Three packets. Connection open. This is called the TCP three-way handshake
and it happens at the start of every TCP connection, every time.

---

### Step 3 — TLS: The Encryption Envelope

After the TCP handshake, the browser and Google negotiated encryption.

Display filter used: `tls`

A packet labelled Client Hello was found and expanded:

```
Transport Layer Security
  TLSv1.3 Record Layer
    Handshake Protocol: Client Hello
      Server Name Indication
        Server name: www.google.com
```

This revealed something worth understanding. Even in an encrypted HTTPS
connection, the domain name is visible in the Client Hello packet before
full encryption is established. Anyone watching the network — an ISP, a
corporate firewall, someone on the same Wi-Fi — can see which website is
being connected to, even though they cannot read the content.

The version negotiated was TLS 1.3, which is the current modern standard.
Older versions (TLS 1.0, 1.1, and the older SSL) have known weaknesses and
are considered insecure.

This is not a flaw specific to this connection. It is a known limitation
of TLS. A newer feature called Encrypted Client Hello (ECH) would hide the
domain name too, but it is not yet widely deployed.

---

### Step 4 — Round Trip Time Graph

From Statistics > TCP Stream Graphs > Round Trip Time, a graph of the
connection's health appeared.

The RTT line was nearly flat, hovering around 0.05 milliseconds, with no
spikes. The session ran from about 0.2s to 0.7s and involved 10 packets
outgoing and 11 returning. A flat RTT line means a stable, healthy
connection. Spikes would indicate congestion, packet loss, or an
overloaded server.

---

### Step 5 — Protocol Hierarchy

From Statistics > Protocol Hierarchy, a breakdown of every protocol in
the capture:

```
Protocol              Packets    Bytes
Frame (total)         167        144,858
Ethernet              167        2,338
Internet Protocol v4  167        3,340
UDP                   14         112       — DNS queries
QUIC IETF             14         18,998    — Google's modern transport
TCP                   153        3,060
TLS                   153        148,120   — encrypted web traffic
```

91.6% of all packets were TLS-encrypted. There was no bare HTTP to Google.
Everything was wrapped in TLS 1.3.

QUIC was also present — this is Google's newer transport protocol that
runs over UDP rather than TCP. It skips the traditional TCP handshake and
is faster as a result. Google uses it for many of its services. It was
not expected to appear and turned out to be one of the more interesting
observations in the session.

### Project 1 conclusion

Visiting google.com produced this sequence: two parallel DNS queries (IPv4
and IPv6), both resolved in under 9ms from a cached local response. Google
returned six IP addresses for load balancing. The browser opened a TCP
connection with a three-way handshake, then negotiated TLS 1.3 encryption.
During that negotiation, the domain name www.google.com was briefly visible
in plain text in the Client Hello packet. The full session produced 167
packets and 144,858 bytes. 91.6% was encrypted. No unencrypted HTTP to
Google was observed. The connection was stable throughout with a flat RTT.

---

## Project 2 — Is Any Traffic Sent in Plain Text?

### The question

Does any device on this network send data in plain text that could be
intercepted and read by someone watching?

### What was done

Captured for 60 seconds with no capture filter, visiting google.com during
the session. Then applied the display filter `http`.

### What appeared

One HTTP stream was found. Right-clicking and selecting Follow > HTTP
Stream showed the full conversation in plain text:

```
Request (browser → server):
GET /success.txt?ipv4 HTTP/1.1
Host: detectportal.firefox.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0)

Response (server → browser):
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 8

success
```

This is Firefox automatically checking whether the network has a captive
portal — the login page sometimes shown at hotels or airports before Wi-Fi
access is granted. Firefox does this in the background every time it
connects to a network. It uses plain HTTP intentionally because HTTPS might
itself be blocked by a captive portal before the user logs in.

A second identical result appeared. Firefox had repeated the same check.
Both results were this routine background request, not user-initiated
browsing.

The filter `ftp or telnet` was also applied. Nothing appeared. FTP and
Telnet are old protocols that transmit passwords in plain text. Finding
them would have been a serious concern. Finding nothing is the correct
result.

### Project 2 conclusion

The only unencrypted HTTP traffic was Firefox's automatic captive portal
detection, which is routine and harmless. No user-initiated browsing data,
cookies, passwords, or form content was transmitted in plain text. No FTP
or Telnet traffic was detected. All actual web browsing used HTTPS and
TLS 1.3.

---

## Project 3 — Can I Map the Network Just by Watching?

### The question

Which devices are active on this local network, and can they be identified
from their traffic alone — without sending a single probe or scan?

### What ARP is

Before a computer can send data to another device, it needs two things:
the IP address (which it usually already knows) and the MAC address (the
hardware identifier burned into the network card). ARP is how a computer
finds the MAC address it needs. It broadcasts a question to every device
on the local network: "Whoever has this IP address — what is your MAC?"

Only the device with that IP replies.

### What was observed

Display filter applied: `arp`

The same message appeared repeatedly:

```
Who has 192.168.161.2? Tell 192.168.161.1

Source MAC:      00:50:56:c0:00:08
Destination MAC: ff:ff:ff:ff:ff:ff  (broadcast — sent to all devices)
Opcode:          request
Sender IP:       192.168.161.1
Target IP:       192.168.161.2
```

The destination `ff:ff:ff:ff:ff:ff` is the broadcast address. It means
"deliver this to every device on the segment." Every device receives it,
but only the one with IP 192.168.161.2 should reply.

From Statistics > Endpoints > Ethernet tab, with the ARP filter active:

```
MAC Address            Packets sent   Packets received
00:50:56:c0:00:08      16             0
ff:ff:ff:ff:ff:ff      0              16
```

The first MAC sent all 16 packets and received none — it was the device
asking the question. The second entry is the broadcast address, not a real
device.

The first three pairs of any MAC address identify the manufacturer. The
prefix `00:50:56` is registered to VMware. This confirmed the device
sending ARP requests was the Kali Linux virtual machine — exactly as
expected.

### Passive IP mapping

From Statistics > IPv4 Statistics, five unique IP addresses appeared
across the full capture session:

```
192.168.161.128   — Kali VM (present in every packet)
74.125.24.100     — Google server (most active)
74.125.24.101     — Google server (second address)
34.107.221.82     — Google Cloud Platform infrastructure
103.174.51.65     — Unknown (appeared once only)
```

The 74.125.24.x addresses matched exactly the DNS answers captured in
Project 1. The 34.107.221.82 address belongs to Google Cloud Platform,
likely serving additional page resources. The address 103.174.51.65
appeared once — possibly a brief background check from a browser extension
or system service. It would be worth looking up with `whois 103.174.51.65`
in a future session.

All five addresses were identified without sending a single probe. This is
what passive monitoring means in practice.

### Project 3 conclusion

16 ARP packets were captured, all from the Kali VM (MAC confirmed as VMware
by OUI lookup). All were broadcast requests to the local gateway. The
endpoint table showed only two entries — the sender and the broadcast
address — confirming a simple local network segment. The full IPv4
statistics revealed five unique IP addresses active during the session,
three of which belonged to Google infrastructure and matched the DNS
responses from Project 1. One address remains unidentified and is noted
for future investigation. All of this was learned without sending any
active scans or probes.

---

## The Server Name Indication Finding

This observation from Project 1 is worth its own section.

When a browser opens an HTTPS connection, the very first message it sends
is called the Client Hello. Inside that message, the domain name is written
in plain text — before encryption is established. This is called Server
Name Indication, or SNI.

What it means practically: anyone watching the network can see which
websites are being visited, even on an encrypted connection. They cannot
read the content. But they can see the destination.

ISPs use this to monitor browsing. Corporate firewalls use it to block
certain sites. It is not a bug or a misconfiguration. It is how TLS
currently works by design.

A future extension called Encrypted Client Hello is intended to fix this
by encrypting the domain name too, but it is not yet widely supported.

---

## Commands Used

```bash

# Check interface names before capturing
ip a

# Capture 20 packets to test the setup
tshark -i eth0 -c 20

# Capture only DNS traffic
tshark -i eth0 -f "port 53" -c 10

# Open Wireshark GUI (no sudo needed after group setup)
wireshark

```

Display filters used inside Wireshark:

```
dns               — DNS queries and responses only
http              — unencrypted HTTP traffic
arp               — ARP broadcasts and replies
tcp.flags.syn==1  — every new TCP connection being opened
tls               — encrypted TLS traffic, including Client Hello
ftp or telnet     — old plaintext protocols (nothing found)
```

---

## What to Do Differently Next Time

Set the capture filter before starting, not a display filter after. A
capture filter keeps the file small and focused from the beginning. A
display filter applied after the fact still contains all the unwanted
traffic in the saved file.

When an unknown IP address appears in the statistics, look it up
immediately in a second terminal with `whois <IP>` rather than noting it
for later.

Save every capture file with a descriptive name — for example
`wireshark_dns_google_apr2026.pcapng` — so it can be reopened and
re-analysed later. The default Wireshark filename tells you nothing about
what was captured.

Follow > TCP Stream is the single most useful feature in Wireshark. It
turns a list of individual packets into a readable conversation. Use it
on any packet that seems interesting before drawing conclusions from the
raw list.

---

## Skills Practised

- Configuring Wireshark to run without root using group permissions
- Setting capture filters to limit what is recorded
- Reading DNS query and response packets and extracting timing information
- Identifying the TCP three-way handshake in a packet list
- Finding TLS Client Hello packets and reading the SNI field
- Following TCP streams to read full conversations
- Using the Protocol Hierarchy statistics to understand a capture at a glance
- Using the Endpoints table and OUI lookup to identify devices by MAC address
- Passively mapping active IP addresses from a capture session
- Distinguishing between routine background traffic and user-initiated traffic

---

## Connection to Previous Labs

Lab 03 scanned Kali and Mint from the outside for open ports. Lab 07
traced the path packets take from these machines to Google. Lab 09 watched
the packets themselves — what they contain, which protocols they use, and
what those protocols reveal about the connection.

The TLS finding in this lab connects directly to the port scanning work
in Lab 03. Port 443 appeared as open in those scans. This lab showed what
actually travels through that port: a TLS handshake that encrypts the
content but leaves the destination visible. Knowing a port is open is one
thing. Knowing what it carries and what it exposes in the process is
the next step.


## Follow-up: Unidentified IP 103.174.51.65

During Project 3, one IP address appeared only once in the capture
statistics and could not be identified from the session data alone. It was
noted as an open finding and investigated afterward.

Four commands were run on Kali to identify it:

```bash

whois 103.174.51.65
dig -x 103.174.51.65
curl https://ipinfo.io/103.174.51.65
nmap -sV -p 80,443 103.174.51.65

```

The results identified the address as belonging to Flarezen Ltd., a
Bangladeshi internet service provider registered with APNIC and located in
Dhaka. The reverse DNS lookup returned no configured hostname, which is
normal for ISP infrastructure. The ipinfo.io query confirmed the
organisation as AS147181, Kafrul, Dhaka Division, Bangladesh. The Nmap
scan found both port 80 and 443 filtered — no public web server running,
consistent with a routing or backbone node rather than a hosted service.

The address most likely belongs to infrastructure sitting between the local
network and the wider internet. A single packet passed through or
originated from that node during the capture session. There is nothing
unexpected here.

Finding status: closed.
