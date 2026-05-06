# Lab 07 – Ping and Traceroute

## What This Lab Is About

In the earlier labs, I found out which machines were on the network and which
ports were open. But I had not yet looked at how data actually travels — from
my machine, through the network, all the way to a destination like Google.

That is what this lab is about.

Two tools helped me do this: ping and traceroute. By the end of this lab, I
could see not just whether a machine was reachable, but how far away it was,
how many stops the data made along the way, and what those stops told me.

---

## A Simple Way to Think About This

Imagine you send a letter to a friend in another city. The letter does not fly
directly to your friend. It goes to your local post office first, then to a
sorting facility, then to another city's facility, then finally to your
friend's door.

Every one of those stops is a hop.

Ping checks whether your friend received the letter and how long the round
trip took. Traceroute shows you every post office the letter passed through
on the way there.

---

## Environment

- Kali Linux: 10.218.101.36, connected via eth0 (wired, inside VMware on
  Windows 11)
- Linux Mint: 10.218.101.76, connected via wlp3s0 (wireless, separate
  physical laptop)
- Gateway: 10.218.101.182 (mobile hotspot)
- Both machines on the same subnet: 10.218.101.0/24

---

## Part One — Ping

### What ping does

Ping sends a small packet to a destination and waits for a reply. It
measures how long the round trip takes, in milliseconds. It also counts
how many packets were sent versus how many came back, which tells you
whether data is being lost along the way.

### The commands I ran

```bash
ping -c 4 127.0.0.1       # ping the machine itself (loopback)
ping -c 4 10.218.101.182  # ping the gateway
ping -c 4 google.com      # ping an external server
ping -c 5 google.com      # same, with five packets instead of four
```

The `-c` flag means count. `ping -c 4` sends exactly four packets and stops
automatically. Without it, ping runs forever until you press Ctrl+C.

---

### Ping results — Kali Linux

**Localhost (127.0.0.1)**

```
4 packets transmitted, 4 received, 0% packet loss
rtt min/avg/max/mdev = 0.023/0.040/0.061/0.014 ms
```

Round-trip time: 0.040 milliseconds on average. This is the machine talking
to itself — the packet never leaves. It is always this fast.

**Gateway (10.218.101.182)**

```
4 packets transmitted, 4 received, 0% packet loss
rtt min/avg/max/mdev = 4.215/4.932/6.135/0.726 ms
```

Round-trip time: about 5 milliseconds. The packet left Kali, reached the
hotspot, and came back. One hop, very close.

**Google (first run, 4 packets)**

```
4 packets transmitted, 4 received, 0% packet loss
rtt min/avg/max/mdev = 66.072/68.184/70.257/2.069 ms
```

Round-trip time: about 68 milliseconds. The server that responded was named
`bkk03s01-in-f14.1e100.net` — this is a Google server located in Bangkok,
Thailand. From Bandarban, Bangladesh, reaching Bangkok in 68ms is normal.

**Google (second run, 5 packets)**

```
5 packets transmitted, 5 received, 0% packet loss
rtt min/avg/max/mdev = 63.985/66.510/69.271/1.849 ms
```

Slightly faster this time. The small difference between runs is normal —
network conditions change moment to moment.

---

### Ping results — Linux Mint

**Localhost (127.0.0.1)**

```
4 packets transmitted, 4 received, 0% packet loss
rtt min/avg/max/mdev = 0.086/0.093/0.115/0.012 ms
```

Still very fast. Slightly slower than Kali's 0.040ms, but this difference
is too small to mean anything.

**Gateway (10.218.101.182)**

```
4 packets transmitted, 4 received, 0% packet loss
rtt min/avg/max/mdev = 3.077/3.518/3.766/0.276 ms
```

About 3.5 milliseconds. Slightly faster than Kali's 5ms to the same
gateway. This makes sense — Linux Mint is a physical machine on wireless,
while Kali is a virtual machine inside Windows. The path is slightly
different even though the destination is the same.

**Google (first run, 4 packets)**

```
4 packets transmitted, 4 received, 0% packet loss
rtt min/avg/max/mdev = 68.687/73.930/77.644/3.576 ms
```

Round-trip time: about 74 milliseconds. Slightly higher than Kali's 68ms
to Google. Both are reaching Google, but Mint connected to a different
Google server — `pnmaaa-az-in-f14.1e100.net` versus Kali's Bangkok server.
Google has many servers and routes each request to whichever one is
available at that moment.

**Google (second run, 5 packets)**

```
5 packets transmitted, 5 received, 0% packet loss
rtt min/avg/max/mdev = 66.990/73.736/83.869/6.280 ms
```

Similar result. The higher mdev (6.280 vs Kali's 1.849) shows that Mint's
wireless connection has slightly more variation in timing than Kali's wired
connection through VMware. This is expected — wireless is always a little
less consistent than wired.

---

### Reading the ping output — what each part means

**time= (round-trip time)**
How long the packet took to reach the destination and come back, measured
in milliseconds. Lower is better. For local machines, under 10ms is normal.
For international servers from Bangladesh, 60–100ms is normal.

**ttl= (time to live)**
Every packet starts with a TTL number, set by the sending machine. Each
router the packet passes through subtracts one. If TTL hits zero, the packet
is discarded. Google's packets arrived at both machines with TTL=114. Since
Google typically starts packets at 128, this means the packet passed through
approximately 14 routers on the way. That number becomes clearer once you
see the traceroute output.

**0% packet loss**
Every packet that was sent came back. This means the connection is clean and
stable. If packets were being dropped, this percentage would be higher.

**mdev (mean deviation)**
How much the response times varied from one packet to the next. A low mdev
means the connection is consistent. A high mdev means there are small
interruptions or fluctuations — common on wireless or congested networks.

---

## Part Two — Traceroute and Tracepath

### What these tools do

Where ping just tells you whether you reached the destination, traceroute
shows you every router your data passed through along the way. Each row in
the output is one hop — one router that forwarded your packet closer to the
destination.

Tracepath and traceroute do the same job with slightly different methods.
Tracepath does not require administrator privileges. Traceroute gives more
detail, especially for later hops.

### Installation notes

Tracepath was not installed on Kali by default. I installed it:

```bash
sudo apt install iputils-tracepath
```

Traceroute was not installed on Linux Mint by default. I installed it:

```bash
sudo apt install traceroute -y
```

Traceroute was already present on Kali (version 2.1.6).

---

### Tracepath results — Kali Linux

```
1?: [LOCALHOST]                      pmtu 1500
 1:  10.218.101.182                  2.963ms
 1:  10.218.101.182                  2.790ms
 2:  no reply
 3:  10.32.202.17                   32.929ms
 4:  no reply
 5:  10.99.249.102                  45.580ms
 6:  123.49.50.33                   30.237ms
 7:  123.49.13.28                   36.097ms
 8:  123.49.8.101                   32.320ms
 9:  123.49.8.103                   31.082ms
10:  123.49.8.216                   58.046ms asymm 9
11:  123.49.8.149                   39.992ms asymm 10
12:  123.49.8.183                   41.525ms asymm 11
13:  142.251.195.122               102.013ms asymm 16
14-30: no reply
     Too many hops: pmtu 1500
```

### Traceroute results — Kali Linux

```
traceroute to google.com (142.251.222.142), 30 hops max
 1  10.218.101.182          11.785 ms
 2  * * *
 3  10.32.202.17            47.951 ms
 4  * * *
 5  10.99.249.102           45.043 ms
 6  123.49.50.33            38.909 ms
 7  123.49.13.28            36.601 ms
 8  123.49.8.101            28.464 ms
 9  123.49.8.103            35.688 ms
10  123.49.8.216            23.913 ms
11  123.49.8.149            44.062 ms
12  123.49.8.183            43.401 ms
13  142.251.195.122         69.646 ms
14  * * *
15  142.251.52.214          73.594 ms
16  192.178.82.238          65.909 ms
17  142.251.239.233         65.656 ms
18  142.251.233.78          75.705 ms
19  142.251.229.251         84.344 ms
20  209.85.253.85           80.307 ms
21  pnmaaa-az-in-f14.1e100.net (142.251.222.142)  73.929 ms
```

### Tracepath results — Linux Mint

```
1?: [LOCALHOST]                      pmtu 1500
 1:  _gateway                         4.957ms
 2:  no reply
 3:  10.32.202.17                    30.861ms
 4:  no reply
 5:  10.99.249.102                   36.206ms
 6:  123.49.50.33                    28.825ms
 7:  123.49.13.28                    36.743ms
 8:  123.49.8.101                    27.982ms
 9:  123.49.8.103                    40.476ms
10:  123.49.8.216                    39.858ms asymm 9
11:  123.49.8.149                    50.297ms asymm 10
12:  123.49.8.183                    36.381ms asymm 11
13:  142.251.195.122                 81.177ms asymm 16
14-30: no reply
     Too many hops: pmtu 1500
```

---

### Reading the traceroute output — what each part means

**Hop 1 — the gateway (10.218.101.182)**

This is always the first stop. Every packet leaving either machine goes
through the mobile hotspot first. From Kali it showed as the IP address
directly. From Mint it showed as `_gateway` — Linux Mint uses a name for
it, but it is the same device.

**Hops 2 and 4 — `* * *` and `no reply`**

These routers did not respond. This does not mean they were missing or
broken. It means they were configured to ignore traceroute probes — a
common security setting on ISP routers. The packet still passed through
them. They just did not announce themselves.

This is the same idea as the firewall behaviour from Lab 03. Silence is
not the same as absence.

**Hop 3 — 10.32.202.17**

This is the first ISP router that did respond. It is using a private IP
address (10.x.x.x), which means it is internal to the ISP's own network
and not visible from the public internet.

**Hops 6 through 12 — 123.49.x.x addresses**

These are routers belonging to the ISP's backbone network. The packet is
moving through the ISP's infrastructure before handing off to Google.

**The `asymm` label in tracepath output**

Asymm stands for asymmetric. It means the return path of the packet took
a different number of hops than the outgoing path. This is normal. Internet
routing is not always the same in both directions — the path from here to
Google and the path from Google back here may go through different routers.

**Hop 13 — 142.251.195.122**

This is the first Google-owned address. The packet has left the ISP's
network and entered Google's infrastructure. This happened at hop 13.

**Hops 14 onward in tracepath — all `no reply`**

Once the packet entered Google's internal network, tracepath could not see
any more hops. Google's internal routers do not respond to tracepath probes.
This is why tracepath reported "Too many hops" — it kept waiting for replies
that never came, eventually giving up at hop 30.

Traceroute handled this better and continued showing hops all the way to
hop 21, which was the final Google server.

**Hop 21 — the destination**

`pnmaaa-az-in-f14.1e100.net (142.251.222.142)` — this is Google's server
that sent back the final reply. The domain `1e100.net` is owned by Google.
The packet made 21 hops to get there.

---

### Comparing Kali and Mint side by side

Both machines followed the same path: gateway → ISP routers → Google. This
makes sense because they share the same gateway and the same internet
connection through the mobile hotspot.

The differences were small but worth noting. Kali's first hop showed the
gateway as an IP address. Mint showed it as `_gateway` — a hostname that
Linux Mint assigned automatically. Both refer to the same device.

Mint's tracepath stopped seeing hops after number 13, just as Kali's did.
Both hit the same wall when entering Google's private network. Traceroute
on Kali continued past that point because it uses a different probing
method that Google's edge routers respond to differently.

The timing numbers were slightly different between the two machines at each
hop — a few milliseconds here and there — because Kali is on a wired
virtual adapter inside Windows while Mint is on a physical wireless card.
The path is the same, but the starting conditions are different.

---

## What I Noticed That Was Not Expected

Tracepath stopped at hop 13 on both machines and reported "Too many hops,"
even though the destination was clearly being reached — the ping results
showed 0% packet loss to Google throughout. At first this looks like a
failure. It is not. It means Google's internal routers do not respond to
the specific probe type that tracepath uses. The tool ran out of visible
hops, but the destination was never unreachable.

Traceroute on Kali continued past hop 13 and reached the destination at
hop 21, showing that the two tools probe differently and one can see
further into certain networks than the other.

The gateway ping from Kali showed noticeably more variation in the second
run (up to 15ms) compared to the first run (maximum 6ms). This kind of
variation in a mobile hotspot connection is normal — hotspot signal strength
and mobile network load fluctuate constantly.

---

## Skills Practised

- Running ping with a fixed packet count and reading every field in the output
- Understanding what TTL, round-trip time, packet loss, and mdev actually mean
- Running tracepath and traceroute to the same destination and comparing results
- Reading `* * *` and `no reply` as meaningful data rather than errors
- Comparing results from two physically different machines on the same network
- Recognising when a tool's limitation is the explanation, not a network problem

---

## Connection to Previous Labs

Lab 01 established the IP addresses on each machine. Lab 02 confirmed which
machines were alive on the network. Lab 03 examined which ports were open
on each machine. Lab 07 now shows what happens between the machines and the
outside world — not just whether a connection exists, but how that connection
is structured, how many steps it takes, and where those steps lead.

The `* * *` hops in this lab are the same concept as the filtered ports in
Lab 03. In both cases, something is there but is not answering. Learning to
read silence correctly is one of the more important habits in this kind of work.

---

## Follow-up: Linux Mint Traceroute (Later Session)

During the original lab session, the traceroute command was not run on
Linux Mint. I ran it in a follow-up session to complete the comparison.

The network addresses had changed by this point because DHCP assigned
new values when the machines reconnected. This is normal behaviour and
expected.

```
Current session addresses:
Mint IP:   192.168.254.76
Kali IP:   192.168.254.36
Gateway:   192.168.254.228
```

### Mint ping results (this session)

Localhost and gateway both responded cleanly with 0% packet loss.

The first Google ping run showed 25% packet loss — one packet out of four
did not come back. The second run with five packets showed 0% packet loss
and consistent times around 38ms. The first run's loss was likely a brief
wireless fluctuation rather than a network problem. Running the test twice
and comparing is good practice.

```
ping -c 4 google.com  →  25% packet loss, avg 54ms  (first run)
ping -c 5 google.com  →  0% packet loss, avg 38ms   (second run)
```

Google's server this session was `maa05s12-in-f14.1e100.net` — a server
in Chennai, India, rather than Bangkok or Pune from previous sessions.
Google routes each connection to whichever server is available at that
moment. The address changes but the behaviour is the same.

### Mint traceroute to Google

```
traceroute to google.com (142.250.67.46), 30 hops max
 1  _gateway (192.168.254.228)   2.530 ms
 2  * * *
 3  10.32.202.1                 26.083 ms
 4  * * *
 5  10.99.249.126               36.256 ms
 6  10.99.249.46                42.020 ms
 7  123.49.50.33                32.633 ms
 8  123.49.13.28                26.201 ms
 9  * * *
10  142.251.201.26              40.174 ms
11  142.251.70.211              45.874 ms
12  142.251.55.217              45.714 ms
13  pnmaaa-bb-in-f14.1e100.net (142.250.67.46)  40.693 ms
```

Destination reached in 13 hops. Mint's path was shorter than Kali's
22-hop result from the original session. This does not mean Mint found a
shorter route — it means Google's load balancing sent the connection to
a closer server this time (Chennai vs Hong Kong).

### Comparing Mint and Kali paths this session

Both machines followed the same ISP backbone in the same order:
gateway → ISP internal routers (10.32.x and 10.99.x) → ISP backbone
(123.49.x) → Google infrastructure (142.251.x) → destination.

The ISP routing infrastructure does not change between sessions even when
local IP addresses do. This confirms that the path your packets take is
determined by your ISP and the destination server, not by your local
network configuration.

Kali reached a Hong Kong server (hkg12s32) in 22 hops at around 63ms.
Mint reached a Chennai server (maa05s12) in 13 hops at around 38ms.
Different servers, different hop counts, both reached through the same
ISP backbone.

### The 25% packet loss on Mint — first ping run

One packet out of four was dropped on the first ping attempt to Google.
The second run of five packets showed no loss at all. This is the correct
way to interpret such a result — a single dropped packet on a wireless
connection during a brief test is not evidence of a network problem. It
is evidence of normal wireless behaviour. Running the test a second time
and getting a clean result confirms the connection is healthy.

A persistent packet loss — say 20% or higher across multiple consecutive
runs — would be worth investigating. One dropped packet in one run is not.
