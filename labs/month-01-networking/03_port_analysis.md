# Network Lab 03 – Port Scanning

## Objective

I wanted to find out which ports are open on the devices in my local network.
But more than that, I wanted to understand what the results actually mean —
including what it means when nothing shows up at all. That second part turned
out to be the most important lesson of this entire lab.

---

## Environment

- Scanner machine: Kali Linux (Dell Latitude 7410, VMware)
- Targets: Linux Mint laptop, Windows 11 host
- Tool used: Nmap 7.95
- Scan types: Default scan, version scan (-sV), skip-ping scan (-Pn),
  full port scan (-p-), SYN scan (-sS)

---

## What Is a Port? (Before I Begin)

Before I ran any scan, I needed to understand what a port actually is.
Let me explain it the way I understood it.

A computer does not just connect to the internet as one big open channel.
Instead, it divides its network connection into thousands of numbered doors
called ports. Each port is assigned to a specific type of service. Port 22
is typically used for SSH — remote login. Port 80 is used for websites.
Port 443 is used for secure websites.

When I run a port scan, I am knocking on each of those doors one by one
and listening for a response. Some doors open. Some are shut. Some have a
guard standing behind them who does not even acknowledge my knock.

A computer has 65,535 ports in total. Nmap's default scan checks the 1,000
most commonly used ones. I can also tell it to check all 65,535 if needed —
and in this lab, I did exactly that.

---

## What I Did — Commands I Ran

### Step 1 — I confirmed which devices were on the network first

```bash
nmap -sn 10.218.101.0/24
```

I had already done this in Lab 02, but I ran it again here to confirm
the four live hosts before starting any port scanning. Same four machines
showed up. Good. I knew where to aim.

### Step 2 — I started with Linux Mint (10.218.101.76)

The lab guide recommended starting here, so I did.

```bash
nmap 10.218.101.76
```

My first attempt. Standard scan — checks the top 1,000 ports and reports
their states. I was expecting to see a few open ports. I got nothing.

```bash
nmap -sV 10.218.101.76
```

I tried again with version detection, thinking maybe the first scan missed
something. Still nothing. All ports filtered.

```bash
nmap -Pn 10.218.101.76
```

I had read that sometimes a machine blocks ping but might still have open
ports. The -Pn flag skips the ping check and scans anyway. I tried this.
Same result.

```bash
nmap -p- 10.218.101.76
```

At this point I wondered if the problem was that I was only scanning 1,000
ports. So I told Nmap to scan all 65,535 ports. This took about 22 minutes
to complete. I waited. Same result.

```bash
nmap -sS -Pn 10.218.101.76
```

My final attempt on Mint. A SYN scan — a quieter method that sends only
half of a connection request and listens for a response without completing
the full handshake. Sometimes called a stealth scan. I hoped this would get
through where the others had not. Same result again.

Five attempts. Five identical results.

### Step 3 — I moved to Windows (10.218.101.121)

```bash
nmap 10.218.101.121
```

One standard scan. Results came back within seconds. Three open ports found.

---

## What Actually Happened — The Raw Output

### Linux Mint — every scan came back like this

Nmap scan report for 10.218.101.76
Host is up (0.0086s latency).
All 1000 scanned ports on 10.218.101.76 are in ignored states.
Not shown: 1000 filtered tcp ports (no-response)
MAC Address: F8:28:19:15:BD:41 (Liteon Technology)

The full 65,535-port scan ran for 22 minutes and returned the exact same
message — all ports filtered, no response on any of them.

Meanwhile, ping worked perfectly throughout the whole experiment:

64 bytes from 10.218.101.76: icmp_seq=1 ttl=64 time=5.84 ms
64 bytes from 10.218.101.76: icmp_seq=2 ttl=64 time=6.72 ms

The machine was clearly on. It was clearly reachable. But it would not
reveal a single port.

### Windows — the very first scan found three open ports

The machine was clearly on. It was clearly reachable. But it would not
reveal a single port.

### Windows — the very first scan found three open ports

Nmap scan report for 10.218.101.121
Host is up (0.00060s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT     STATE SERVICE
902/tcp  open  iss-realsecure
912/tcp  open  apex-mesh
5357/tcp open  wsdapi
MAC Address: A4:6B:B6:24:6E:22 (Intel Corporate)


---

## What I Learned — Making Sense of the Results

### First, the three port states I encountered

**Open** means a service is running on that port and it responded to my
probe. Something is actively listening there. This is the most important
state because it shows where a machine can actually be reached.

**Closed** means the port exists and the host responded, but no service is
running on it right now. The machine acknowledged my knock and said there
is nothing here at the moment.

**Filtered** means the port gave no response at all. A firewall or security
rule is silently dropping my probe before it even reaches the port. The
machine is not saying open or closed — it is simply not answering.

### What I eventually understood about Linux Mint

When the first scan came back with nothing, I thought
I had done something wrong. When the second scan came back the same way,
I started questioning whether Nmap was working correctly. By the time I
finished the fifth scan and got the same result, I finally accepted what
the output was actually telling me.

The machine was protected by a firewall that was dropping every single TCP
port probe silently. But here is the part that confused me at first — ping
was still working. How can a machine respond to ping but refuse to respond
to a port scan?

The answer is that ping and port scanning use completely different types of
network traffic. Ping uses a protocol called ICMP. Port scanning uses TCP.
A firewall can be configured to allow ICMP through while blocking all TCP
probes at the same time. Linux Mint was doing exactly that.

The way I think about it now is this. My neighbour is definitely home —
I can see the lights on. But every time I walk up and knock on any door
or window, a security guard quietly steps in front of me and turns me away
without saying a word. The neighbour is there. The guard just does not let
any messages through. That guard is the Linux firewall.

Once I understood this, I stopped seeing the result as a failure. A machine
with all ports filtered is not broken or offline. It is protected. It is
actually a sign of a well-configured system.

### What the three Windows ports told me

**Port 902** is used by VMware Workstation. When VMware is installed and
running, it opens this port for communication. VMware is installed on my
Windows machine, so this made complete sense.

**Port 912** is also VMware-related. It handles internal management
communication between VMware components. Again, expected.

**Port 5357** is used by a Windows feature called Web Services Discovery,
or WSD. This is a built-in Windows service that helps devices on a local
network find each other — printers, shared folders, other networked devices.
Windows turns this on by default. I had not deliberately opened it, but it
was there because Windows opened it on its own.

---

## What If Both Machines Had Shown Nothing?

I thought about this during the lab, and I think it is worth recording.

If both machines had returned all-filtered, my first instinct might have
been to assume the scans were broken. But that would have been the wrong
response. A machine that shows all-filtered is not necessarily turned off
or empty. It might be running dozens of services internally, none of which
are exposed to the network. Many real-world servers are configured exactly
this way.

The right response would have been to verify with ping that the machines
were still online, check that I was scanning the correct subnet, and try
approaching from a different method. The lesson from Lab 02 still applies
here: a clean scan result that shows nothing does not guarantee nothing is
there. Absence of evidence is not evidence of absence.

---

## Thinking Like an Analyst — What the Results Actually Mean

After finishing the scans, I sat with the results and asked myself the
questions the lab guide suggested.

**Which ports are open?**

On Linux Mint, none. Not a single port visible out of 65,535.
On Windows, three: 902, 912, and 5357.

**Why are they open?**

VMware is installed and running on Windows, which explains 902 and 912.
Windows enables network discovery by default, which explains 5357. These
were all expected once I knew what to look for.

**Are they expected?**

Yes. When you know what software is installed on a machine, you can predict
which ports should be open. When the results match expectations, that is
reassuring. When they do not match, that is when you need to investigate.

**Are they risky?**

Linux Mint presents no visible attack surface. Nothing is reachable from
the network. That is excellent security posture.

Windows presents a small attack surface. The VMware ports are low risk on
a private trusted network but would need closer attention on a public one.
Port 5357 leaks information about what is shared on the machine and is worth
disabling if network discovery is not actually needed.

The honest truth about this lab is that I came in expecting a tidy list
of open ports to analyse. What I got instead was one completely invisible
machine and one machine with three ports. That outcome was more educational
than the expected one would have been. It taught me that all-filtered is
not an empty result — it is the most common result on a well-managed
real-world network. And it taught me that learning to read silence as
meaningful data is one of the most important habits in security work.

---

## What I Think I Did Right

I want to record this because it is easy to feel like a lab failed when
the main target shows nothing at all.

I did not stop after the first scan showed nothing. I tried five different
methods before drawing any conclusion. I used ping independently to confirm
the machine was definitely online. I scanned both targets so I could compare
a silent machine against an exposed one side by side. I followed the
recommended scanning order from the lab guide. And when the all-filtered
result kept appearing, I treated it as data rather than dismissing it as
a mistake.

That persistence — trying multiple approaches before concluding anything —
is exactly how this kind of work is supposed to be done.

---

## Skills I Practised

- Running multiple types of Nmap scans and understanding what each one does
- Reading and interpreting filtered, closed, and open port states
- Understanding the difference between ICMP (ping) and TCP (port) traffic
- Identifying known services by their port numbers
- Treating unexpected results as data rather than errors
- Applying analyst thinking to scan output after the fact

---

## How This Connects to the Previous Labs

Lab 01 established IP addresses by reading each machine's own network
configuration. Lab 02 confirmed those addresses from the outside using
a ping sweep. Lab 03 goes one level deeper.

It does not just ask which machines are alive. It asks which doors on
those machines are open, what is behind them, and what that means.

The same four hosts from Lab 02 appear again here. But I am no longer
just looking at them as dots on a network map. I am looking at them as
machines with services, defences, and attack surfaces. That shift in
thinking is what this lab was really about.

---

## What Comes Next

**Lab 04 — Service Enumeration**

Linux Mint is locked down and has nothing visible to work with yet.
Windows has three exposed ports — 902, 912, and 5357. The next lab will
go deeper on those services: what exact versions are running, what they
allow, and whether any known vulnerabilities are associated with them.

Before starting Lab 04, there is a useful optional exercise I plan to try.
Installing and starting SSH on Linux Mint, then scanning it again, would
show me what it looks like when a port deliberately opens on a previously
silent machine. Seeing the before and after would make the concept much
more concrete.

---

*Network Lab 03 · Port Scanning · April 2026 · 24-Month Cybersecurity Roadmap · Week 2*
