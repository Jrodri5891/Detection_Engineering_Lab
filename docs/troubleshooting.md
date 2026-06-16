# Troubleshooting

Real issues hit while building the lab and how they were resolved. Most trace
back to the Universal Forwarder not delivering data to the Splunk indexer.

## Forwarder data not reaching Splunk ("host not receiving")

Work through these in order.

### 1. Confirm the indexer is reachable from the endpoint

```powershell
Test-NetConnection 192.168.56.1 -Port 9997
```

`TcpTestSucceeded : True` means the path to the indexer's receiving port is
open. If it's `False`, it's almost always the firewall — go to step 2.

### 2. Rule out the host firewall

Temporarily disable the firewall just to confirm it's the cause:

```powershell
Set-NetFirewallProfile -All -Enabled False
```

If data starts flowing with the firewall off, **don't leave it off.** Turn it
back on and add a scoped inbound rule for the receiving port instead:

```powershell
Set-NetFirewallProfile -All -Enabled True

New-NetFirewallRule -DisplayName "Splunk 9997 Inbound" -Direction Inbound `
  -Protocol TCP -LocalPort 9997 -Action Allow -Profile Any
```

The Host-Only network sits on the **Public** firewall profile, so the rule
must apply to all profiles (`-Profile Any`) — a Private/Domain-only rule won't
take effect.

### 3. Check the forwarder service account

A common cause of missing Windows event data: the **SplunkForwarder** service
isn't running under an account that can read the event logs.

- Open `services.msc`
- Find **SplunkForwarder** -> **Properties** -> **Log On** tab
- Set it to **Local System account**, then restart the service

### 4. Read the forwarder log

```powershell
Get-Content "C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log" -Tail 200
```

Look for connection errors to the indexer (TcpOutputProc / cooked connection
failures) or input-parsing problems.

## Index name mismatch (data silently missing)

Splunk index names are **case-sensitive**. `index = Sysmon` in `inputs.conf`
will not land in the `sysmon` index — the events simply disappear with no
error. Use lowercase `sysmon`.

## PowerShell logging not captured

If T1059.001 PowerShell activity isn't showing up, confirm the PowerShell
Operational input is enabled in the forwarder's `inputs.conf`:

```ini
[WinEventLog://Microsoft-Windows-PowerShell/Operational]
disabled = false
index = windows
```

## Kali connectivity verification

After placing Kali on the Host-Only network, predetermine and verify its IP and
DNS before running any attacks:

```bash
ip a                  # confirm the host-only interface and IP (192.168.56.103)
nslookup <target-ip>  # reverse lookup against the DC
nslookup <hostname>   # forward lookup for lab.local hosts
```

If name resolution fails, point Kali's DNS at the domain controller
(`192.168.56.10`).
