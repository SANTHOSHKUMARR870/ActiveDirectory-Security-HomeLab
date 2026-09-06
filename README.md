# ActiveDirectory-Security-HomeLab
# Active Directory SOC Home Lab

A simulated enterprise network built to practice red team attack simulation and blue team detection engineering, using Splunk as a centralized SIEM.

This project follows the workflow of a real security operations environment: build a domain-joined network, simulate real attacker techniques mapped to MITRE ATT&CK, and detect them through log analysis.

---

## Architecture

![Network Diagram](soc%20-%20project(Network%20diagram).png)

| Component | Role | IP |
|---|---|---|
| Windows Server 2022 | Domain Controller (`sandy.local`) | 192.168.10.7 |
| Windows 10 | Domain-joined client / attack target | DHCP |
| Splunk Enterprise (Ubuntu Server) | Centralized SIEM | 192.168.10.10 |
| Kali Linux | Attacker machine | 192.168.10.250 |

All machines run on an isolated NAT Network (`192.168.10.0/24`) inside VirtualBox, keeping attack traffic contained away from the host network.

---

## What I Built

- **Active Directory Domain Services** — promoted a Windows Server to a Domain Controller, created Organizational Units (IT, HR), and added domain user accounts
- **Domain-joined Windows 10 client** — configured to authenticate against the DC via DNS
- **Endpoint logging pipeline** — installed Sysmon (using Olaf Hartong's MITRE ATT&CK-aligned config) and the Splunk Universal Forwarder on both the DC and the Windows 10 client
- **Centralized SIEM** — Splunk Enterprise running on Ubuntu Server, indexing logs from all domain-joined hosts

![Active Directory Users and Computers](soc%20-%20AD_users.png)

---

## Attacks Simulated

### 1. RDP Brute Force & Password Spray

Used Hydra against the Windows 10 client's RDP service, first attempting a traditional brute force (many passwords, one account), then a password spray (one password, many accounts) to demonstrate account lockout evasion.

```bash
hydra -l rmohan -P passwords.txt rdp://192.168.10.100 -t 1 -w 5
hydra -L users.txt -p 'p@ssw0rd1!' rdp://192.168.10.100 -t 1 -w 5
```

![Brute force attack result](soc%20-%20BruteForce_Attack.png)

![Password spray attack result](soc%20-%20passwordspray%20Attack.png)

**Detection in Splunk:**

```
index=endpoint (EventCode=4625 OR EventCode=4624)
| stats count by Account_Name, EventCode
```

Results showed 22 failed logon attempts (Event Code 4625) spread across multiple accounts, and 1 successful logon (Event Code 4624) — the exact signature that distinguishes a password spray from a single-account brute force.

![Brute force and spray detection in Splunk](soc%20-%20SIEM%20detection.png)

---

### 2. Credential Dumping — MITRE ATT&CK T1059.001

Simulated a Mimikatz-style credential dumping technique using Atomic Red Team, executed via PowerShell on the Windows 10 client.

```powershell
Invoke-AtomicTest T1059.001 -TestNumbers 1
```

**Detection in Splunk:**

Sysmon (Event ID 1) captured the process creation event and automatically tagged it with the corresponding MITRE ATT&CK technique, thanks to the Olaf Hartong Sysmon config:

```
technique_id=T1059.001, technique_name=PowerShell
CommandLine=powershell.exe ... Invoke-Mimikatz -DumpCreds
```

![Mimikatz detection with MITRE tagging](soc%20-%20Mimikatz%20detection.png)

This confirmed the full detection chain: attacker technique → Sysmon telemetry → automatic MITRE ATT&CK tagging → searchable in Splunk.

---

## Troubleshooting Highlights

Most of the actual time on this project went into diagnosing issues, not building — which mirrors real IT/SOC work. A few worth documenting:

- **Crowbar's RDP module threw a persistent "not enough arguments for format string" error** — a known bug in that Kali package version. Switched to Hydra instead, after first manually verifying RDP connectivity with `rdesktop` to isolate whether the issue was the tool or the target.
- **Network Level Authentication (NLA) silently blocked brute-force attempts**, causing instant "connection failed" errors with no useful message. Disabled NLA temporarily on the Windows 10 client's Remote Desktop settings to allow the older RDP handshake these tools expect.
- **A single typo in `inputs.conf`** (`disables = false` instead of `disabled = false`) silently disabled Security log forwarding for hours, with no error anywhere — only caught by manually reviewing the forwarder's own internal logs.
- **Windows Defender's cloud-delivered protection blocked and deleted the Atomic Red Team Mimikatz test script**, even with a folder exclusion already in place — resolved by temporarily disabling real-time and cloud-delivered protection on the isolated lab VM.
- **DHCP silently reassigned the Windows 10 client's IP** between sessions, breaking previously-working attack commands — a good reminder to verify target IPs before assuming a tool or technique has failed.

---

## Key Takeaways

This project made the theory behind detection engineering concrete. Reading about Event Code 4625 is very different from watching a real brute-force attack generate that exact pattern in your own SIEM seconds after you launch it. It also reinforced that most real security work is methodical troubleshooting — checking logs, isolating variables, verifying assumptions — rather than any single dramatic step.

---

## Tools Used

VirtualBox · Windows Server 2022 · Windows 10 · Kali Linux · Splunk Enterprise · Splunk Universal Forwarder · Sysmon (Olaf Hartong config) · Hydra · Atomic Red Team · MITRE ATT&CK

---

## Reference

This lab follows the structure of the [myDFIR Active Directory Home Lab](https://www.youtube.com/playlist?list=PLGrcVHQv6mp-_5jY1XU1SJ3tjoUMrTI7E) series, rebuilt independently with additional password spraying and extended troubleshooting documentation.
