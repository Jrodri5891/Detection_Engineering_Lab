# Detections

Splunk detections authored for this lab, mapped to MITRE ATT&CK. Each `.spl`
file in `splunk/` contains the saved-search logic plus a documented header
(technique, data source, schedule, and trigger condition).

| Detection | ATT&CK | Data Source | File |
|-----------|--------|-------------|------|
| PowerShell Execution (Atomic Red Team) | T1059.001 | Sysmon EID 1 | [`splunk/PurpleTeam_AtomicRedTeam_Event-Detection.spl`](splunk/PurpleTeam_AtomicRedTeam_Event-Detection.spl) |
| Failed Logon / Brute Force | T1110 / T1078 | Security EID 4625 | [`splunk/PurpleTeam_Failed_Logon_Detection.spl`](splunk/PurpleTeam_Failed_Logon_Detection.spl) |
| Network Logon | T1021 | Security EID 4624 | [`splunk/PurpleTeam_Network_Logon_Detection.spl`](splunk/PurpleTeam_Network_Logon_Detection.spl) |
| Network Service Discovery | T1046 | Sysmon EID 3 / Firewall | [`splunk/PurpleTeam_Network_Recon_Detection.spl`](splunk/PurpleTeam_Network_Recon_Detection.spl) |
| Network Share Enumeration | T1135 | Security 5140/5145 | [`splunk/PurpleTeam_Share_Enumeration_Detection.spl`](splunk/PurpleTeam_Share_Enumeration_Detection.spl) |

All detections run on a 5-minute cron schedule, trigger when results > 0, and
fire a Discord webhook alert to a SOC channel.
