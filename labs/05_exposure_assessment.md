# Lab 05 – Exposure Assessment of a Windows Host in a Local Network

## Objective

In Lab 04, I found out what services were running behind the three open
ports on my Windows machine. I had names, versions, and behaviour. But
knowing what is running is only half the job. The more important question
is this — could any of these services become a problem?

This lab is about learning to look at normal system services and ask
whether they introduce risk. Not exploiting anything. Not assuming the
worst. Just thinking carefully about what is exposed, how it is exposed,
and what that means.

This kind of thinking has a name in security work. It is called exposure
assessment.

---

## Environment

- Scanner machine: Kali Linux (Dell Latitude 7410, VMware)
- Target: Windows 11 – 10.218.101.121
- Tool used: Nmap 7.95
- Known services going in: ports 902, 912, and 5357 from Lab 04

---

## Executive Summary

A local network assessment was conducted on a Windows 11 host at
10.218.101.121. Three network-accessible services were identified across
previous labs. No critical vulnerabilities were detected during automated
scanning. However, one service was found to use unencrypted communication,
which introduces risk in any environment where the network cannot be
fully trusted. All findings are consistent with expected behaviour given
the software installed on the machine.

---

## What Is Exposure Assessment?

In Lab 03, I asked — which doors are open?
In Lab 04, I asked — what is behind each door?
In Lab 05, I am asking — should these doors be open at all, and what
happens if the wrong person walks through?

That progression is how real security analysis works. Each question builds
on the last one. Finding a service is not the end. Understanding its risk
is.

A useful way to think about it: imagine a shop with three windows left
open overnight. The question is not just which windows are open. It is —
what can someone see through each one? What could they reach? What would
they learn just by looking?

---

## Scope

- Target: Windows 11 – 10.218.101.121
- Network: Local subnet (10.218.101.0/24)
- Ports assessed: 902, 912, 5357
- Method: Passive and safe scanning only — no exploitation attempted

---

## Methodology

The assessment followed the same progression used across all previous
labs. Hosts were first discovered, then ports were scanned, then services
were enumerated, and finally the behaviour and configuration of those
services was evaluated for risk. In this lab, the focus shifted from
collecting information to interpreting it.

One additional scan was run specifically for this lab:

```bash
nmap --script=vuln -p 902,912,5357 10.218.101.121
```

This runs Nmap's vulnerability detection scripts against the three known
ports. These scripts do not attack anything. They check for known patterns,
common misconfigurations, and signatures that match publicly documented
issues. The purpose is to see whether anything obvious surfaces
automatically before doing manual analysis.

---

## Raw Output — Vulnerability Script Scan

Starting Nmap 7.95 at 2026-04-23 07:10 EDT
Nmap scan report for 10.218.101.121
Host is up (0.00044s latency).
PORT     STATE SERVICE
902/tcp  open  iss-realsecure
912/tcp  open  apex-mesh
5357/tcp open  wsdapi
MAC Address: A4:6B:B6:24:6E:22 (Intel Corporate)

No vulnerability signatures were detected. The three ports remained open
and the services responded normally, but the scripts found nothing that
matched known vulnerability patterns.

This is a common result and an important one to understand. A clean
vulnerability scan does not mean a system is perfectly secure. It means
no automated tool found a match. Manual analysis, version research, and
configuration review still matter.

---

## Findings

### Finding 1 — VMware Authentication Service (Port 902)

**Service:** VMware Authentication Daemon 1.10
**Communication:** Encrypted with SSL
**Accessible from network:** Yes — confirmed from Kali in Lab 04

VMware Workstation opened this port as part of its normal operation.
The service handles authentication between VMware components. Version
1.10 uses SSL, which means communication through this port is encrypted.
That is a positive sign. Intercepting this traffic would be significantly
harder than intercepting unencrypted traffic.

**Risk:** Moderate. The service is reachable from other machines on the
local network, which increases attack surface. In a home lab with trusted
devices, this is acceptable. In a shared or public network, a remote
authentication service exposed to the network would require much closer
attention.

**Assessment:** Acceptable in this controlled environment. Worth monitoring
if the network changes.

---

### Finding 2 — VMware Authentication Service (Port 912)

**Service:** VMware Authentication Daemon 1.0
**Communication:** No SSL — unencrypted
**Accessible from network:** Yes — confirmed from Kali in Lab 04

This is the most significant finding in this lab.

Port 912 runs an older version of the same VMware Authentication Daemon
running on port 902. The critical difference is that this version does not
use SSL. That means any data passing through this port travels in plain
text across the network. If another device on the same network were
capturing traffic — a technique called sniffing — the contents of that
communication could be read.

In my home lab, with only my own machines on the network, the practical
risk is very low. But the pattern itself is important. Unencrypted
authentication services are exactly the kind of thing that gets flagged
in real security assessments. Credentials, session data, or internal
communication sent without encryption is a risk regardless of how trusted
the environment feels.

**Risk:** Moderate to high depending on environment. On an untrusted or
shared network, this would be a serious concern.

**Assessment:** This is the most interesting finding from a learning
perspective. It demonstrates that two services can look nearly identical
— same software, adjacent ports, similar function — but have meaningfully
different security postures based on one configuration difference.

---

### Finding 3 — Windows Network Discovery Service (Port 5357)

**Service:** Microsoft HTTPAPI 2.0 (SSDP/UPnP)
**Communication:** HTTP
**Accessible from network:** Yes — confirmed from Kali in Lab 04

Windows opened this port itself as part of its built-in network discovery
feature. It allows devices on the local network to find each other —
printers, shared folders, media devices, and so on. When a connection was
attempted through this port in Lab 04, the service responded with
"Service Unavailable," meaning it is listening but only responds to
specific Windows protocols, not general web requests.

The main concern with this service is information disclosure. A device
probing this port can learn that the machine is running Windows, what
network features are enabled, and potentially what is shared on the
machine. That information is useful for reconnaissance — building a picture
of the target before attempting anything else.

**Risk:** Low to moderate. The service itself is unlikely to be directly
exploited in a standard home lab. But it leaks information, and
information is what makes further analysis possible.

**Assessment:** Expected default Windows behaviour. Low concern in this
environment. Worth disabling if network discovery is not actually needed.

---

## Overall Risk Evaluation

The Windows host does not present critical vulnerabilities based on
automated scanning. All services found match the software known to be
installed on the machine. Nothing unexpected appeared.

However, three things are worth carrying forward as awareness:

First, all three services are reachable from other machines on the
network. They are not just local processes. They have a network presence,
which means they have an attack surface.

Second, port 912 operates without encryption. Any environment where
network traffic could be intercepted would make this service a priority
for investigation or remediation.

Third, port 5357 leaks system information by design. That is not a flaw
— it is what the service is built to do. But it means this port actively
helps anyone scanning the machine build a more complete picture of it.

---

## Recommendations

These are not urgent actions for a home lab. They are the kind of
decisions that would need to be made in a real environment.

- Restrict access to ports 902 and 912 to trusted hosts only, using
  firewall rules that block connections from unknown machines.
- Investigate whether port 912 can be configured to use SSL, or whether
  the service can be replaced by the encrypted version on port 902.
- Disable port 5357 and Windows network discovery if the feature is not
  actively used. Default-on services that expose system information should
  be turned off when they serve no operational purpose.
- Monitor network-exposed services regularly, particularly those handling
  authentication, for unexpected behaviour or connection attempts.

---

## Limitations

This assessment was limited to what is visible from one machine on the
local network using safe, non-exploitative scanning techniques. No
authenticated scanning was performed — meaning the scans had no login
access to the target system and could only observe from the outside.
No exploitation was attempted. Version numbers were detected by Nmap
but not independently verified against vendor release histories. A full
professional assessment would include manual research, authenticated
access, and cross-referencing detected versions against known
vulnerability databases such as CVE records.

---

## Real-World Relevance

In enterprise environments, unencrypted authentication services like the
one on port 912 are a recognised risk pattern. On a corporate network
with many devices, a single machine capturing traffic — an attack called
a man-in-the-middle — could intercept credentials or session data from
any unencrypted service within range. Port 5357-style information
disclosure is also commonly used in the early reconnaissance phase of
real attacks, where the goal is to learn as much as possible about a
target before deciding how to proceed. Neither of these is hypothetical.
Both appear regularly in incident reports and penetration testing findings.

---

## Analyst Thinking — The Questions I Answered

**Which services increase the attack surface?**

All three, because all three are reachable from the network. Ports 902
and 912 are more significant because they involve authentication.

**Which service leaks information?**

Port 5357, by design. It exists to help devices find each other, which
means it broadcasts details about the machine.

**Which service is least secure in its current configuration?**

Port 912. Unencrypted communication on an authentication service is the
clearest security weakness found in this assessment.

**Which finding is most worth investigating further?**

Port 912. The absence of encryption on an authentication service is
exactly the kind of configuration detail that separates a secure
deployment from a vulnerable one.

---

## What I Did Right

I did not run the vulnerability scan and then declare the system safe
just because nothing came back. A clean automated scan is not a clean
bill of health. It is the starting point for manual thinking.

I used the results from previous labs rather than starting over. The
value of keeping structured notes across labs is that each one feeds
the next. The version numbers from Lab 04 meant I already knew what to
analyse before this lab began.

I paid attention to the difference between port 902 and port 912. They
look almost identical at first glance — same software, similar function,
adjacent port numbers. The encryption difference is easy to miss if you
are reading output quickly. Slowing down to compare them properly was
where the most useful finding came from.

---

## Skills Practised

- Running vulnerability detection scripts with Nmap --script=vuln
- Interpreting a clean scan result correctly — as a starting point, not
  a conclusion
- Evaluating services individually for exposure, encryption, and risk
- Writing findings in a structured format with descriptions, risk levels,
  and assessments
- Distinguishing between services with similar function but different
  security posture
- Producing recommendations based on findings rather than just
  describing what was found

---

## How This Connects to the Previous Labs

Lab 01 found my machine on the network.
Lab 02 found the other devices.
Lab 03 found which ports were open.
Lab 04 found what services were behind those ports.
Lab 05 asked whether those services matter from a risk perspective.

Each lab answered a question that the previous one raised. 
The findings only made sense because the earlier steps had already been done. 

---

## What Comes Next

The natural next step is learning how to research services and versions
against known vulnerability databases. When a scan reveals that a service
is running version 1.0 of something, the question becomes — has anyone
found a weakness in version 1.0? Is there a public record of it? That
kind of research is what connects enumeration to actual vulnerability
analysis, and it is where the CVE database and tools like Searchsploit
become relevant.

---

