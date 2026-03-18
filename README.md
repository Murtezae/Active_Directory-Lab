# Active Directory Home Lab

A fully functional on-premises Active Directory environment built from scratch for cybersecurity learning. This lab covers Active Directory setup, Splunk log ingestion, brute force attack simulation, and threat detection using Atomic Red Team.

---

## Table of Contents

- [Lab Diagram](#lab-diagram)
- [Environment Overview](#environment-overview)
- [Part 1 — Network Setup & Splunk Installation](#part-1--network-setup--splunk-installation)
- [Part 2 — Sysmon & Splunk Universal Forwarder](#part-2--sysmon--splunk-universal-forwarder)
- [Part 3 — Active Directory Configuration](#part-3--active-directory-configuration)
- [Part 4 — Brute Force Attack with Kali Linux](#part-4--brute-force-attack-with-kali-linux)
- [Part 5 — Atomic Red Team](#part-5--atomic-red-team)
- [Key Takeaways](#key-takeaways)

---

## Lab Diagram

![Lab Diagram](/images/Diagram.png)

The lab uses a NAT network (`192.168.10.0/24`) with four virtual machines running in VirtualBox.

---

## Environment Overview

| Machine | OS | IP Address | Role |
|---|---|---|---|
| Splunk Server | Ubuntu Server 22.04 | 192.168.10.10 | Log aggregation (Splunk Enterprise) |
| Active Directory | Windows Server 2022 | 192.168.10.7 | Domain Controller (ADDC01) |
| Target Machine | Windows 10 | 192.168.10.100 | Endpoint with Splunk UF + Sysmon |
| Attacker | Kali Linux | 192.168.10.250 | Offensive tooling |

**Domain:** `mydfir.local`

---

## Part 1 — Network Setup & Splunk Installation

Configured a static IP on the Ubuntu Splunk server via Netplan, then installed Splunk Enterprise and enabled it to start on boot:
```bash
sudo dpkg -i splunk-<version>-linux-amd64.deb
cd /opt/splunk/bin
sudo ./splunk start
sudo ./splunk enable boot-start -user splunk
```

On the Windows 10 target machine, set a static IP of `192.168.10.100` via Network Adapter settings with the gateway at `192.168.10.1` and DNS initially pointing to `8.8.8.8` (updated to `192.168.10.7` after Active Directory was configured).

---

## Part 2 — Sysmon & Splunk Universal Forwarder

Installed the Splunk Universal Forwarder on the Windows 10 target machine pointing to the Splunk server at `192.168.10.10:9997`. Installed Sysmon using the [olafhartong/sysmon-modular](https://github.com/olafhartong/sysmon-modular) config for richer endpoint telemetry.
```powershell
cd C:\Users\m\Downloads\Sysmon
.\Sysmon.exe -i ..\sysmonconfig.xml
```

Created `inputs.conf` under `C:\Program Files\SplunkUniversalForwarder\etc\system\local\` to define which logs to forward:
```ini
[WinEventLog://Application]
index = endpoint
disabled = false

[WinEventLog://Security]
index = endpoint
disabled = false

[WinEventLog://System]
index = endpoint
disabled = false

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = endpoint
disabled = false
renderXml = true
source = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
```

> **Note:** The Splunk Forwarder service must be restarted after updating `inputs.conf` and should run as Local System Account to have permission to read Security logs.

In the Splunk web UI, created an index named `endpoint` to match the config above, then enabled receiving on port `9997`. This same setup was repeated on ADDC01. With both machines forwarding logs, searching `index="endpoint"` confirmed the full pipeline was working.

![Data in Splunk](/images/splunk.png)

---

## Part 3 — Active Directory Configuration

On ADDC01, used Server Manager to install the Active Directory Domain Services role, then promoted the server to a Domain Controller with a new forest:

- **Domain:** `mydfir.local`

Created two Organizational Units and two domain users to simulate a real environment:

| OU | User | Username |
|---|---|---|
| IT | Jenny Smith | jsmith |
| HR | Terry Smith | tsmith |

Updated the Windows 10 machine's DNS to point to the domain controller (`192.168.10.7`) and joined it to `mydfir.local` via System Properties. Verified by logging in as `jsmith`.

---

## Part 4 — Brute Force Attack with Kali Linux

Configured Kali Linux with a static IP of `192.168.10.250`, enabled RDP on the Windows 10 target for both domain users, then set up the attack:
```bash
# Install crowbar
sudo apt-get install -y crowbar

# Build a wordlist from rockyou (top 20 + known target password appended)
head -n 20 /usr/share/wordlists/rockyou.txt > passwords.txt
```

Ran the brute force attack against `jsmith`:
```bash
hydra -t 4 -l jsmith -P passwords.txt rdp://192.168.10.100
```

Hydra successfully found the valid credential.

![Hydra brute force success](/images/hydra.png)

### Detection in Splunk

Searched for `jsmith` activity within the last 15 minutes:
```
index=endpoint jsmith
```

| Event ID | Description | Count |
|---|---|---|
| 4625 | Failed logon | 25 |
| 4624 | Successful logon | 2 |

The 25 failed attempts map directly to the passwords in `passwords.txt`. Multiple 4625s in rapid succession from the same source IP is a clear brute force indicator. Expanding the 4624 event also revealed the source workstation name and IP pointing directly back to the Kali machine.

![Splunk brute force detection](/images/eventcode.png)

---

## Part 5 — Atomic Red Team

Added a Windows Defender exclusion for the C drive to prevent Atomic Red Team files from being removed, then installed:
```powershell
Set-ExecutionPolicy Bypass CurrentUser
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing)
Install-AtomicRedTeam -getAtomics -Force
```

### Test: Create Local Admin User (T1136.001)

Ran the MITRE ATT&CK technique for **Create Account: Local Account**, which simulates an attacker creating a new local administrator account named `NewLocalUser`:
```powershell
Invoke-AtomicTest T1136.001
```

Searched Splunk for the created account:
```
index=endpoint NewLocalUser
```

Splunk returned 12 events confirming the account creation was captured via Windows Security logs.

![NewLocalUser in Splunk](/images/image5.png)

---

## Key Takeaways

- **Sysmon dramatically improves endpoint visibility** — without it, many process creation and network events would go unlogged.
- **Brute force attacks are noisy** — multiple Event ID 4625s in a short window from the same source IP is a reliable detection signal.
- **Atomic Red Team exposes visibility gaps** — if a technique generates no Splunk events, that is a gap to investigate, not a sign nothing happened.
- **DNS misconfiguration prevents domain joins** — the target machine's DNS must point to the Domain Controller before attempting to join.

---

## Tools & References

- [VirtualBox](https://www.virtualbox.org/)
- [Splunk Enterprise (Free Trial)](https://www.splunk.com/en_us/download/splunk-enterprise.html)
- [Splunk Universal Forwarder](https://www.splunk.com/en_us/download/universal-forwarder.html)
- [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [sysmon-modular config by olafhartong](https://github.com/olafhartong/sysmon-modular)
- [Atomic Red Team](https://github.com/redcanaryco/invoke-atomicredteam)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [Crowbar](https://github.com/galkan/crowbar)
- [draw.io](https://app.diagrams.net/)

