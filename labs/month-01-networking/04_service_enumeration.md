# Network Lab 04 – Service Enumeration

## Objective

In Lab 03, I found three open ports on my Windows machine. But finding
an open port is only the beginning. The real question is — what is actually
running behind that port? What service is it? What version? Is it something
I recognise?

That is what this lab was about. Not just finding open doors, but knocking
on each one and asking — who is in there, and what are you doing?

---

## Environment

- Scanner machine: Kali Linux (Dell Latitude 7410, VMware)
- Target: Windows 11 – 10.218.101.121
- Tool used: Nmap 7.95
- Known open ports going in: 902, 912, 5357

---

## What Is Service Enumeration?

Port scanning tells you which doors are open. Service enumeration tells
you what is standing behind each door.

Imagine you are a security guard doing rounds in a building. In the last
round, you found three unlocked doors. Now you are going back to each one
and asking — What is this room used for? Who has access? Is anything
sensitive inside?

That follow-up visit is an enumeration. It turns a list of open ports into
actual intelligence about the system.

---

## What I Did — Step by Step

### Step 1 — I confirmed the open ports first

Before going deeper, I ran the same basic scan from Lab 03 to confirm
the three ports were still open.

```bash
nmap 10.218.101.121
```

PORT       STATE   SERVICE

902/tcp    open    iss-realsecure

912/tcp    open    apex-mesh

5357/tcp   open    wsdapi

Still there. Good. Now I could start asking what they actually were.

### Step 2 — I asked Nmap to detect service versions

```bash
nmap -sV 10.218.101.121
```

This flag tells Nmap not just to say a port is open, but to probe the
service running on it and try to identify exactly what it is and which
version it is running.

This is what came back:

PORT       STATE     SERVICE           VERSION

902/tcp    open      ssl/vmware-auth   VMware Authentication Daemon 1.10 (Uses VNC, SOAP)

912/tcp    open      vmware-auth       VMware Authentication Daemon 1.0 (Uses VNC, SOAP)

5357/tcp   open      http              Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)

Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Now the ports had names. Real names. Port 902 and 912 were both running
VMware Authentication Daemons — different versions of the same VMware
service. Port 5357 was running Microsoft's HTTP API, used for device
discovery on the local network.

And Nmap had also figured out the operating system — Windows. That was
a bonus piece of information I did not even ask for.

### Step 3 — I added default scripts to learn even more

```bash
nmap -sV -sC 10.218.101.121
```

The -sC flag runs a set of built-in scripts that come with Nmap. These
scripts do not just detect a service — they actually talk to it and see
how it responds. Think of it as moving from knocking on a door to having
a short conversation with whoever answers.

This is what the scripts added:

5357/tcp open  http                  Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)

|_http-title: Service Unavailable

|_http-server-header: Microsoft-HTTPAPI/2.0

The script tried to access port 5357 like a web browser would. The
service responded with "Service Unavailable." That means the port is
open and the service is running, but it is not serving any content. It
is listening without talking back. That is still useful information.

### Step 4 — I focused only on the three known ports

```bash
nmap -p 902,912,5357 -sV -sC 10.218.101.121
```

Instead of scanning all 1,000 ports again, I told Nmap to look only at
the three I already knew about. This made the scan faster and the output
cleaner. Same results, but more focused.

This is a habit worth building early — once you know which ports matter,
stop scanning everything and concentrate on those.

### Step 5 — I ran safe scripts for even deeper detail

```bash
nmap --script=default,safe -p 902,912,5357 10.218.101.121
```

This ran a wider collection of safe scripts against the three ports.
Safe means the scripts will not do anything harmful or disruptive — they
only observe and report. Here is what came back that was new:

902/tcp open  iss-realsecure

|_banner: 220 VMware Authentication Daemon Version 1.10: SSL Required,...

912/tcp open  apex-mesh

|_banner: 220 VMware Authentication Daemon Version 1.0, ServerDaemonPr...

The scripts grabbed the banners from ports 902 and 912. A banner is a
message that a service broadcasts when something connects to it. It
usually announces what it is and what version it is running. This
confirmed what the -sV scan had already told me — VMware Authentication
Daemon, versions 1.10 and 1.0.

There was also this result from the scripts:

| dns-blacklist:

|   SPAM

|     list.quorum.to - SPAM

One of the scripts checked whether the IP address appeared on any
spam or blacklists. The result said SPAM for one list. This does not
mean my machine is sending spam. In a home lab using a mobile hotspot,
shared IP address ranges can sometimes appear on these lists. It is
worth noting but not alarming in this context.

---

## What I Learned — Making Sense of Each Service

### Port 902 — VMware Authentication Daemon 1.10

This port is opened by VMware Workstation when it is running. Version
1.10 uses SSL, which means communication through this port is encrypted.
The service handles authentication between VMware components — it is how
different parts of VMware confirm they are allowed to talk to each other.

It is open because VMware is installed and running on my Windows machine.
On a private home network, this is low risk. On a public or exposed
network, it would be a surface worth investigating because remote
management services are attractive targets.

### Port 912 — VMware Authentication Daemon 1.0

This is a slightly older version of the same type of service on port 902.
The key difference is that this one does not use SSL — the connection is
not encrypted. That means any communication on this port travels in plain
text, which is worth knowing even in a home lab.

Both ports exist because VMware opens them as part of its normal
operation. I did not configure them manually. VMware put them there.

### Port 5357 — Microsoft HTTPAPI 2.0 (SSDP/UPnP)

This port is opened by Windows itself as part of its network discovery
features. It allows devices on the local network to find each other
automatically — printers, shared drives, media players, and so on.

The scripts showed that when something tries to connect to this port like
a browser would, the service returns "Service Unavailable." That means it
is listening, but only for specific types of requests from specific
Windows protocols. It is not a web server in the normal sense. It is a
background service doing a specific Windows job.

---

## What the Results Told Me When I Put It All Together

After running all five scans, here is what I actually knew about the
Windows machine:

- It is running Windows, confirmed by Nmap's OS detection.
- VMware Workstation is installed and actively running, with two of its
  authentication services exposed on the network.
- Windows has its built-in network discovery service open on port 5357.
- None of these services were surprising given what I know is installed
  on the machine.
- Port 912 does not use encryption, which is a small flag worth
  remembering for later labs.

---

## Analyst Thinking — The Questions I Asked After

**Which services are running?**

Port 902 — VMware Authentication Daemon 1.10, encrypted with SSL.

Port 912 — VMware Authentication Daemon 1.0, no encryption.

Port 5357 — Microsoft HTTPAPI 2.0, used for Windows network discovery.

**Why are they running?**

VMware is installed and running on the Windows host. VMware opens these
ports itself as part of normal operation. Port 5357 is opened by Windows
by default when network discovery is enabled.

**Are they expected?**

Yes. Every service found matches what is known to be installed on the
machine. When expectations match results, that is reassuring. When
something unexpected appears, that is when investigation becomes urgent.

**Are they risky?**

On a private home lab network, the risk is low. In a different context,
each service would need more scrutiny. Port 912 without encryption is the
most noteworthy detail from this lab — unencrypted remote services can
expose credentials or data if someone is listening on the network.

The important rule I am carrying forward from this lab is this: the goal
of enumeration is understanding, not action. Finding a service does not
mean attacking it. It means learning what it is, why it is there, and
whether it belongs.

---

## What I Did Right

Running the basic scan first before going deeper meant I always had a
clean baseline to compare against. Moving through the scans in steps —
from basic to version to scripts to focused — meant each scan built on
the previous one rather than jumping straight to the most complex option.
Grabbing the banners gave me confirmation from the services themselves,
not just from Nmap's guesses. And stopping to ask the analyst questions
at the end meant I left the lab with actual understanding, not just a
collection of terminal output.

---

## Skills I Practised

- Using -sV to detect service versions beyond just port states
- Using -sC to run default scripts and extract extra information
- Focusing scans on specific ports with -p for cleaner results
- Using --script=default, safe to grab banners and deeper service details
- Reading and interpreting banner output from live services
- Connecting open ports to the software installed on the target machine
- Applying analyst thinking to service-level results

---

## How This Connects to the Previous Labs

Lab 01 found my IP address. Lab 02 found the other devices on the
network. Lab 03 found which ports were open. Lab 04 answered the next
question — what is actually running on those ports?

Each lab has been one step closer to understanding the machine rather
than just knowing it exists. That progression matters. In real security
work, nobody stops at "I found an open port." The open port is just the
beginning of the question.

---

## What Comes Next

**Lab 05 — Basic Vulnerability Awareness**

Now that I know what services are running and which versions they are,
the next step is asking whether those versions have any known weaknesses.

Now, I am thinking of the shift — from "what is running" to "what risks does that carry" —
is the foundation of how real vulnerability assessment works.

---

 
