# Lab Setup Guide

End-to-end build notes for the Purple Team Detection Engineering Lab. The lab
runs entirely on a single host in VirtualBox and produces real Windows/Sysmon
telemetry into Splunk, against which detections are engineered and tested.

## Architecture

| Host | Role | OS | IP (host-only) |
|------|------|----|----------------|
| **DC01** | Domain Controller (AD DS + DNS) | Windows Server 2022 | 192.168.56.10 |
| **WIN11** | Domain-joined endpoint (telemetry source) | Windows 11 Pro | 192.168.56.20 |
| **Kali** | Attacker / adversary emulation | Kali Linux | 192.168.56.103 |
| **Splunk** | Indexer + search head | Splunk Enterprise (on host) | 192.168.56.1:9997 |

- **Domain:** `lab.local` (NetBIOS `LAB`)
- **Networking:** VirtualBox **Host-Only** adapter `192.168.56.0/24` for lab
  traffic, plus a **NAT** adapter on each VM for internet (tooling, updates).
- **Indexes:** `sysmon` (Sysmon events) and `windows` (Security/System logs).

## Prerequisites

- VirtualBox with a Host-Only network (`192.168.56.0/24`).
- ISOs: Windows Server 2022, Windows 11, Kali Linux.
- Splunk Enterprise (free Developer license) installed on the host.
- Splunk Universal Forwarder + Sysmon (SwiftOnSecurity config) for the endpoint.

## 1. Domain Controller (DC01)

1. Create the VM (type *Windows 2022*, 4 GB RAM). **Remove the unattended-install
   floppy** before first boot — leaving it causes a *"cannot find Microsoft
   Software License Terms"* error. Install manually.
2. Set a static IP `192.168.56.10` on the Host-Only adapter; DNS = `127.0.0.1`.
3. Install the **AD DS** role and promote to a new forest: `lab.local`.
4. Create OUs (**HR, IT, Finance**) and populate test users, including a
   Domain Admin and a standard user (e.g. `rgonzalez`).

## 2. Endpoint (WIN11)

1. Create the VM as type **Windows 11** (this auto-enables **TPM 2.0 + EFI**,
   required by the installer). **Skip** the unattended install.
2. At setup, skip the product key and choose **Windows 11 Pro**. For a local
   account, run `start ms-cxh:localonly` at the OOBE.
3. Point DNS at the DC (`192.168.56.10`) so domain join can resolve `lab.local`.
   (Temporarily disable the NAT adapter during the join if needed.)
4. Join the domain `lab.local`.
5. Install **Sysmon** with the SwiftOnSecurity config for rich process,
   network, and file telemetry.
6. Install the **Splunk Universal Forwarder** and point it at the indexer
   (`192.168.56.1:9997`). Configure inputs (see below).

## 3. Splunk (host)

1. Install **Splunk Enterprise** on the host; enable receiving on **9997**.
2. Create indexes **`sysmon`** and **`windows`**.
3. Install the **Splunk Add-on for Microsoft Windows** and the
   **Splunk Add-on for Sysmon** (provides field extractions / CIM mapping).

### Forwarder inputs

`C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`

```ini
[WinEventLog://Security]
disabled = false
index = windows

[WinEventLog://System]
disabled = false
index = windows

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = false
index = sysmon
renderXml = true
```

> **Index names are case-sensitive.** `index = Sysmon` will silently fail to
> index — it must be lowercase `sysmon` to match the index created in Splunk.

## 4. Attacker (Kali)

1. Create the Kali VM on the Host-Only network (`192.168.56.103`).
2. Confirm connectivity and DNS resolution against the DC.
3. Tooling used: `nmap`, `enum4linux`, `smbclient`, and **Atomic Red Team**
   (run locally on the endpoint for adversary emulation).

## 5. Detections & Alerting

- Five Splunk saved-search alerts (prefix `PurpleTeam_`) — see
  [`../detections/`](../detections/) for the detection-as-code.
- Each runs on a 5-minute cron, triggers when results > 0, and fires a
  **Discord webhook** alert into a SOC channel.

## Validation checklist

- [ ] `index=sysmon` returns Sysmon EID 1 events from `WIN11.lab.local`.
- [ ] `index=windows EventCode=4625` returns failed logons.
- [ ] An Atomic Red Team T1059.001 test triggers the PowerShell detection.
- [ ] A detection firing posts a message to the Discord channel.

## Key gotchas

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| WS2022 license-terms error at install | VirtualBox unattended-install floppy | Remove the floppy; install manually |
| Forwarder can't reach Splunk on 9997 | Host firewall blocked the port (Host-Only is on the *Public* profile) | Allow 9997 inbound on **all** profiles |
| Sysmon data not indexing | Case-mismatched index name in `inputs.conf` | Use lowercase `sysmon` |
| Domain join fails to resolve `lab.local` | Endpoint DNS not pointing at the DC | Set DNS to `192.168.56.10` |

Firewall rule used:

```powershell
New-NetFirewallRule -DisplayName "Splunk 9997 Inbound" -Direction Inbound `
  -Protocol TCP -LocalPort 9997 -Action Allow -Profile Any
```
