<img width="779" height="742" alt="image" src="https://github.com/user-attachments/assets/923e63c8-51d2-4a00-b06c-61b4beed75a2" />

# Active Directory Attack & Detection Lab

*(Splunk + Sysmon + Kali Linux + Atomic Red Team)*

Credits: avulman (original concept). This repository documents my implementation of the lab with a focus on detection visibility, identity telemetry, and repeatable security testing.

This project builds a controlled Active Directory environment and generates realistic authentication telemetry (successful and failed logons, brute-force activity, endpoint events) for investigation in Splunk. The goal is to simulate enterprise identity behavior in a contained lab environment.

**Estimated completion time:** 3–4 hours

---

## Overview

This lab connects identity to security operations.

Instead of only standing up Active Directory, this project shows how authentication activity appears in logs, how brute-force attacks generate detectable signals, and how SIEM ingestion + endpoint monitoring combine to create visibility.

The focus is on:

* Identity-based authentication telemetry
* Centralized log aggregation
* Detection validation
* Controlled adversary simulation

---

## Lab Environment

Four virtual machines are deployed using Oracle VM VirtualBox:

* **Windows Server 2022** – Domain Controller (AD DS + DNS)
* **Windows 11** – Domain-joined target endpoint
* **Ubuntu Server 22.04** – Splunk Enterprise
* **Kali Linux** – Attacker machine

All VMs are connected to a shared NAT network.

---

## Network Configuration

Example addressing scheme:

* Splunk Server (Ubuntu): `192.168.10.10`
* Domain Controller: `192.168.10.7`
* Windows 10 Target: `192.168.10.100`
* Kali Linux: `192.168.10.250`
* NAT Gateway: `192.168.10.1`

All machines must share the same VirtualBox NAT Network.

---

## Architecture

```

+-------------------+        +--------------------+
| Kali Linux        |        | Splunk Enterprise  |
| 192.168.10.250    |        | Ubuntu 192.168.10.10
| (Crowbar / ART)   |<-----> | Web:8000  Recv:9997|
+---------+---------+        +----------+---------+
          |                              ^
          | RDP Brute Force              | Forwarded logs
          v                              |
+-------------------+        +----------+---------+
| Windows 11 Target |<------>| Windows Server 2022 |
| 192.168.10.100    |  DNS   | 192.168.10.7        |
| Domain-joined     |  + AD  | AD DS + DNS (DC)    |
+-------------------+        +----------------------+
```

---

## What Gets Logged

Windows systems forward:

* Application logs
* Security logs
* System logs
* Sysmon operational logs

In Splunk, this enables detection of:

* Failed logons (Event ID 4625)
* Successful logons (Event ID 4624)
* RDP authentication patterns
* Endpoint process creation and network activity (Sysmon)

---

## Phase 1 – Splunk Setup (Ubuntu)

1. Configure static IP via netplan.
2. Install Splunk Enterprise.
3. Enable boot-start.
4. Enable receiving on port `9997`.
5. Create an index named `endpoint`.

Verify access via:
`http://192.168.10.10:8000`

---

## Phase 2 – Endpoint Log Forwarding

On both Windows 11 and Windows Server:

1. Install Splunk Universal Forwarder.
2. Set receiving indexer to `192.168.10.10:9997`.
3. Install Sysmon with a configuration file.
4. Configure `inputs.conf` at:

`C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`

Example:

```
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

Restart the Splunk Forwarder service and verify ingestion:

```
index=endpoint
```

---

## Phase 3 – Active Directory Configuration

### Promote Domain Controller

* Add Active Directory Domain Services role.
* Create new forest (example: `demodomain.local`).
* Reboot.

### Create Identity Structure

* Create OUs (example: IT, HR).
* Create domain users (example: jsmith, tsmith).

---

## Phase 4 – Join Windows 10 to Domain

1. Set DNS to Domain Controller (`192.168.10.7`).
2. Join domain.
3. Log in with domain credentials.

Verify domain login works correctly.

---

## Phase 5 – Brute Force Simulation (Kali + Crowbar)

Enable Remote Desktop on Windows 11 target.

From Kali:

```
sudo apt install crowbar
crowbar -b rdp -u tsmith -C passwords.txt -s 192.168.10.100/32
```

This generates failed authentication attempts (Event ID 4625) and possibly a successful login (Event ID 4624).

---

## Detection in Splunk

Confirm ingestion:

```
index=endpoint
```

Failed logons:

```
index=endpoint EventCode=4625
```

Successful logons:

```
index=endpoint EventCode=4624
```

Detect brute force pattern:

```
index=endpoint EventCode=4625
| stats count as failures by host, Account_Name, Source_Network_Address
| sort - failures
```

High counts of 4625 within short time windows indicate brute-force behavior.

---

## Atomic Red Team Validation

Atomic Red Team is used to simulate additional attack techniques and validate telemetry visibility.

Process:

1. Install Atomic Red Team on Windows 11.
2. Execute selected technique tests.
3. Confirm events appear in Splunk.
4. Document any detection gaps.

The goal is validation — not exploitation.

---

## Troubleshooting Checklist

* Ensure Windows 10 DNS points to the Domain Controller.
* Confirm Splunk receiving is enabled on port 9997.
* Restart Splunk Forwarder after modifying inputs.conf.
* Ensure index name matches configuration.
* Verify all VMs share the same NAT network.

---

## Repository Structure

```
README.md
configs/
  inputs.conf
  sysmonconfig.xml
splunk/
  searches.md
screenshots/
  ingestion-proof.png
  4625-spike.png
  4624-success.png
```

---

## Key Learning Outcomes

* How AD authentication events surface in SIEM platforms
* How brute-force attacks appear in event logs
* How endpoint telemetry supports detection engineering
* How identity infrastructure ties directly into security operations
* How to validate detection coverage using controlled adversary simulation

---

## Safety Notice

This lab is for controlled environments only.
Do not run brute-force tools against systems without explicit authorization.


Just tell me which direction you want this repo to lean.
