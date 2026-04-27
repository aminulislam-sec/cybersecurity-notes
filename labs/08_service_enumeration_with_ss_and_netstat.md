# Lab 08 – Finding What Is Listening: Network Service Auditing with ss and netstat

## What This Lab Is About

In the previous labs, I looked at the network from the outside — which
machines were alive, which ports were open, how data travelled to Google.

This lab looked from the inside.

Instead of asking "what can I see from another machine," I asked: what is
my own machine doing right now? Which services are running? Which ones are
listening for incoming connections? And are they listening in a way that
any machine on the network can reach them, or only the machine itself?

This is called a service audit. It is something worth doing regularly on
any machine you are responsible for.

---

## A Simple Way to Think About This

Imagine your house has many doors and windows. Some are facing the street —
anyone walking past can knock on them. Some are facing your private back
garden — only you can reach them from inside.

Every service running on a computer that listens for connections is like one
of those doors or windows. The question this lab asks is: which ones face
the street, and which ones face the garden?

The tool that shows you the answer is called ss. An older version of the
same tool is called netstat. Both do the same job.

---

## Environment

- Kali Linux: 10.218.101.36, wired via eth0, inside VMware on Windows 11
- Linux Mint: 10.218.101.76, wireless via wlp3s0, separate physical laptop
- Gateway: 10.218.101.182 (mobile hotspot)
- Both on subnet: 10.218.101.0/24

---

## The Main Command and What Its Flags Mean

```bash
sudo ss -tulnp
```

This one command does most of the work in this lab. Here is what each
letter means:

`-t` — show TCP connections. TCP is the protocol used by most services like
web servers, databases, and SSH.

`-u` — show UDP connections. UDP is used by services like DNS and some
network discovery tools.

`-l` — show only listening ports. This filters out active conversations and
shows only the doors that are open and waiting for someone to knock.

`-n` — show numbers instead of names. Without this flag, the tool tries to
translate port 22 into the word "ssh" and IP addresses into hostnames. With
it, you see the raw numbers, which is more precise.

`-p` — show the process. This tells you which program opened each port and
what its process ID number is.

The older version of this command is:

```bash
sudo netstat -tulnp
```

It uses the same flags and produces similar output. Both were used in this
lab.

---

## Part One — Kali Linux

### What the scan found

```
Netid  State   Local Address:Port  Process
tcp    LISTEN  127.0.0.1:5432      postgres (pid=1067)
tcp    LISTEN  [::1]:5432          postgres (pid=1067)
```

And from netstat, which caught an additional service:

```
Proto  Local Address      State   PID/Program
tcp    127.0.0.1:5432     LISTEN  1067/postgres
tcp    0.0.0.0:22         LISTEN  3257/sshd
tcp6   :::22              LISTEN  3257/sshd
tcp6   ::1:5432           LISTEN  1067/postgres
```

Two services were running: PostgreSQL (a database) and sshd (which allows
remote login over SSH).

### What the two addresses mean

**127.0.0.1** is the loopback address. It means "this machine only." No
other device on the network can reach a service bound here, regardless of
any firewall settings. PostgreSQL was bound here, which means it is
correctly configured — it accepts connections from Kali itself, and nothing
else.

**0.0.0.0** means "all interfaces." A service bound here is reachable from
any machine on the network. SSH was bound here, which means any machine on
the subnet could attempt to connect to Kali on port 22.

### The established connection

```
udp  10.218.101.36:68  →  10.218.101.182:67  ESTABLISHED  NetworkManager
```

Port 68 is the DHCP client port. Port 67 is the DHCP server port. This
line is showing Kali's network manager maintaining its IP address lease
with the gateway. It is expected and normal.

### Kali security summary

Port 5432 (PostgreSQL) is on 127.0.0.1 — no one on the network can reach
it. This is correct.

Port 22 (SSH) is on 0.0.0.0 — any machine on the network can attempt a
connection. For a personal lab machine this is acceptable. On a machine
facing the public internet, this would need tightening.

---

## Part Two — Linux Mint

### What the scan found

```
Netid  State   Local Address:Port       Process
udp    UNCONN  0.0.0.0:48238           avahi-daemon (pid=867)
udp    UNCONN  127.0.0.54:53           systemd-resolve (pid=715)
udp    UNCONN  127.0.0.53:53           systemd-resolve (pid=715)
udp    UNCONN  224.0.0.251:5353        brave (pid=2199)
udp    UNCONN  0.0.0.0:5353            avahi-daemon (pid=867)
udp    UNCONN  [::]:45137              avahi-daemon (pid=867)
udp    UNCONN  [::]:5353               avahi-daemon (pid=867)
tcp    LISTEN  127.0.0.53:53           systemd-resolve (pid=715)
tcp    LISTEN  127.0.0.1:631           cupsd (pid=1102)
tcp    LISTEN  127.0.0.54:53           systemd-resolve (pid=715)
tcp    LISTEN  [::1]:631               cupsd (pid=1102)
```

Mint had more services running than Kali. This is not surprising. Mint is
a desktop operating system meant for everyday use — it runs printing
services, local DNS, and network discovery tools in the background without
the user doing anything. Kali is built for security testing and starts far
fewer background services by default.

### What each service is

**systemd-resolve on 127.0.0.53 and 127.0.0.54** — this is the local DNS
resolver. When you type a web address, this service translates it into an
IP address. It is bound to localhost only, so it is not accessible from
the network.

**cupsd on 127.0.0.1:631** — this is the printing service (CUPS stands for
Common Unix Printing System). It is bound to localhost only. No network
exposure.

**avahi-daemon on 0.0.0.0:5353** — this is the one that faces the street.
Avahi is a service discovery tool. It broadcasts the machine's hostname
and available services across the local network so that other devices can
find it — printers, shared folders, that sort of thing. Because it is
bound to 0.0.0.0, any machine on the same network can see these
broadcasts. On a trusted home or lab network this is low risk. On a public
or shared network it would reveal information about the machine that you
might prefer to keep private.

**Brave browser on 224.0.0.251:5353** — the browser was open during this
scan and was using mDNS, a multicast protocol used for local network
discovery. The address 224.0.0.251 is a multicast group address, not a
regular IP. It does not expose the machine to the whole network — only to
devices that have joined the same multicast group.

### The established connections (Brave browser was open)

```
tcp  10.218.101.76:46910  →  18.97.36.63:443    brave
tcp  10.218.101.76:34528  →  142.251.43.133:443  brave
tcp  10.218.101.76:45610  →  3.173.21.63:443     brave
tcp  10.218.101.76:53814  →  163.181.201.183:443  brave
tcp  10.218.101.76:40050  →  104.18.39.21:443    brave
udp  10.218.101.76:43638  →  142.250.77.99:443   brave (QUIC)
udp  10.218.101.76:68     →  10.218.101.182:67   NetworkManager (DHCP)
```

All the port 443 connections are HTTPS — the browser talking to websites
over encrypted connections. The UDP port 443 connection is QUIC, which is
a newer protocol that modern browsers use to load pages faster. It uses
UDP instead of TCP but is still fully encrypted. The DHCP line is the same
as on Kali — the machine maintaining its IP address lease.

Listening ports show what your machine is offering to the world. Established
connections show what your machine is doing right now. Both matter.

### Mint security summary

Most services on Mint are localhost-only and not visible from the network.
The only service with genuine network exposure is avahi-daemon. On this
private lab network it is not a problem. On a public network it would be
worth disabling.

To disable it if needed:

```bash
sudo systemctl disable avahi-daemon --now
```

---

## Part Three — Cross-Machine Testing with netcat

Knowing what ss reports is useful. Proving it from another machine is
better.

Netcat (nc) is a simple tool that tries to make a connection to a specific
IP address and port. If it succeeds, the port is open and reachable. If it
fails, either nothing is listening there, or a firewall is blocking it.

These tests were run from Linux Mint, pointing at Kali (10.218.101.36):

```bash
nc -zv 10.218.101.36 22     # SSH
nc -zv 10.218.101.36 5432   # PostgreSQL
nc -zv 10.218.101.36 80     # HTTP
nc -zv 10.218.101.36 9999   # Unknown port
```

The `-z` flag means "scan only, do not send data." The `-v` flag means
"verbose — tell me what happened."

### Results

**Port 22 — connection succeeded.** SSH is listening on 0.0.0.0 on Kali,
so Mint could reach it. This matched what ss reported.

**Port 5432 — connection refused.** PostgreSQL is on 127.0.0.1 on Kali,
so Mint could not reach it at all. The nc test proved that the localhost
binding actually works. Seeing it refused from outside confirms the
configuration is doing what it is supposed to do.

**Port 80 — connection refused.** No web server is running on Kali. Nothing
is listening there, so nothing answered. This is the expected result.

**Port 9999 — connection succeeded.** This was not expected. Nothing in
the ss output showed anything listening on port 9999. But nc reached it.

---

## The Unexplained Finding — Port 9999

This is the most important result from the entire lab, even though it is
also the most unresolved one.

A port was reachable from another machine but did not appear in the service
scan. There are a few possible explanations. Something may have started
listening on that port between the time ss was run and the time nc was run.
A previous test or experiment may have left a listener running. The ss scan
may have been run without sudo, which means it would only show processes
owned by the current user and would miss anything running as root.

The commands to investigate are:

```bash
sudo ss -tulnp | grep :9999
sudo lsof -i :9999
ps aux | grep nc
history | grep 9999
```

In a real security audit, a result like this does not get dismissed. It
gets written down and followed up. An unexplained open port on a machine
you are responsible for is a finding, not a footnote.

---

## The Timing Gap — Why ss and nc Did Not Agree on Port 22

When ss was first run on Kali, SSH did not appear in the output. But nc
from Mint confirmed port 22 was open and accepting connections.

This is a good lesson about how these tools work. ss takes a snapshot of
the current state. Services start and stop. If SSH started after ss was
run but before nc was tested, it would appear open to nc but not in the
ss output.

The fix is simple: always run ss immediately before running nc tests, save
the output with a timestamp, and compare. If nc reaches a port that was
not in your saved snapshot, that gap is worth investigating.

```bash
# Save a timestamped snapshot before testing
sudo ss -tulnp > /tmp/pre_test_$(date +%Y%m%d_%H%M%S).txt
```

---

## Errors That Happened and What They Taught Me

**Running nc with a placeholder instead of a real IP.**
The command `nc -zv TARGET_IP 22` was run literally, with the text
`TARGET_IP` instead of an actual address. The error said "Unknown host."
The fix is to always replace placeholder values before running a command.

**Running source ~/.bashrc inside zsh.**
Kali's default shell is zsh, not bash. The .bashrc file contains
bash-specific commands. Running it inside zsh produced a stream of errors
about unknown commands. The fix is to add aliases to ~/.zshrc when working
in Kali's default shell, or to open a bash shell explicitly before sourcing
a bash config file.

**Using a PID from the wrong machine.**
The investigation template used PID 715 from Mint's output. On Kali,
PID 715 belongs to a completely different process, or may not exist at all.
Process ID numbers are specific to each machine and each session. Always
look up the PID from your own machine's current output before using it in
a command.

---

## Saving a Baseline and Checking for Changes

A baseline is a saved record of what was running at a specific point in
time. Once you have one, you can compare future scans against it to see
what changed.

```bash
# Save today's baseline
sudo ss -tulnp > ~/services_baseline_$(date +%Y%m%d).txt

# Later, compare against it
sudo ss -tulnp | diff - ~/services_baseline_$(date +%Y%m%d).txt
```

If the diff output is empty, nothing changed. If there are lines starting
with `+` or `-`, a service started or stopped since the baseline was saved.

This is one of the simplest and most practical security habits a person can
build. You cannot notice something unexpected if you do not know what
"normal" looks like.

---

## Kali and Mint Side by Side

Kali had two listening services: PostgreSQL on localhost, and SSH on all
interfaces. Its attack surface was small and mostly deliberate.

Mint had more listeners, all but one confined to localhost. The exception
was avahi-daemon, which broadcasts to the local network. Mint also had
eight established outbound connections from the browser, which were all
encrypted HTTPS traffic.

The reason Mint had more services is that it is a desktop operating system
for daily use. It runs printing, local DNS resolution, and network
discovery in the background. Kali is stripped down for testing work and
does not start those services unless you ask it to.

Neither machine had anything alarming running — with the exception of port
9999 on Kali, which remains unexplained.

---

## Skills Practised

- Running ss and netstat to list listening ports and active connections
- Reading the difference between 0.0.0.0 and 127.0.0.1 and knowing why it matters
- Tracing a port back to the process that opened it using the PID
- Using netcat to confirm findings from another machine
- Saving a timestamped baseline and understanding how to diff against it
- Treating an unexplained result as a finding rather than ignoring it
- Recognising when an error in the output is caused by the wrong shell or
  the wrong machine context

---

## Connection to Previous Labs

Lab 03 scanned Kali and Mint from the outside and found that Mint showed
no open ports at all, while Kali showed three. This lab looked at the same
machines from the inside and found a different picture — Mint has several
services listening, all but one on localhost, which is exactly why Lab 03
saw nothing from the network. A machine with all its services bound to
127.0.0.1 looks completely closed to an outside scanner. From the inside,
it is doing quite a lot.

This is the same idea that appeared in Lab 03: silence from the outside
does not mean nothing is happening. It just means nothing is exposed.
This lab showed what was actually running behind that silence.


***Update
Port 9999 follow-up: subsequent investigation found no listener,
no matching process, and no command history. Assessed as a transient
event, likely a brief VMware or lab process. Closed.
