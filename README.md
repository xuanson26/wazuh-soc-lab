**Tech stack:** Wazuh · Sysmon · Shuffle · TheHive · VirusTotal · MITRE ATT&CK

## Overview
This project implements an automated SOC workflow that detects malicious endpoint activity, enriches the alert with threat intelligence, opens a case for investigation, and notifies the analyst, with no manual intervention required during triage.

A Windows 10 endpoint running **Sysmon** and the **Wazuh agent** forwards telemetry to a **Wazuh Manager**. When a custom rule detects **Mimikatz**, the alert is sent to **Shuffle** (SOAR), enriched with **VirusTotal**, created as an alert in **TheHive**, and emailed to the SOC analyst.

## Objectives

- Deploy and configure Wazuh (SIEM/XDR) and TheHive (case management)
- Collect enriched endpoint telemetry using Sysmon
- Detect credential dumping activity with a custom Wazuh rule
- Automate alert enrichment, ticketing, and notification with Shuffle
- Extend detection coverage and map it to MITRE ATT&CK

## Architecture


**Workflow**

1. The Windows 10 client generates events. Sysmon captures process creation, network connections, and other activity.
2. The Wazuh agent forwards these events to the Wazuh Manager.
3. The manager evaluates events against detection rules. A match triggers an alert.
4. The alert is forwarded to Shuffle through a webhook integration.
5. Shuffle extracts the file hash (SHA256) and queries VirusTotal for its reputation.
6. Shuffle creates an alert in TheHive with the enrichment results.
7. Shuffle emails the SOC analyst with the alert details for a response decision.

## Lab Environment

| Component | Platform | Role |
|---|---|---|
| Wazuh Server | Ubuntu 24.04.5 | Wazuh Manager, Indexer, Dashboard |
| TheHive Server | Ubuntu 24.04.5 | TheHive, Cassandra, Elasticsearch |
| Windows Client | Windows 10 | Sysmon, Wazuh Agent |
| Shuffle | Shuffle Cloud | SOAR platform |
| Attacker (extensions) | Kali Linux | Attack simulation |

**Hosting:** [DigitalOcean / VirtualBox / VMware]
**Networking:** [Cloud firewall / Host-only / NAT]

## Implementation

### 1. Wazuh Server
- Installed Wazuh using the official installation assistant
- Retrieved credentials and accessed the dashboard over HTTPS
- Opened the required ports (1514, 1515, 55000, 443)

Details: [`02-wazuh-setup`](02-wazuh-setup/)

### 2. TheHive Server
- Installed dependencies (Java, Cassandra, Elasticsearch) and TheHive
- Configured Cassandra (`cassandra.yaml`), Elasticsearch (`elasticsearch.yml`), and TheHive (`application.conf`)
- Fixed file ownership and permissions on TheHive directories
- Verified services and accessed the web console on port 9000

Details: [`03-thehive-setup`](03-thehive-setup/)

### 3. Windows Client, Sysmon & Wazuh Agent
- Installed Sysmon with the SwiftOnSecurity configuration
- Deployed the Wazuh agent and registered it with the manager
- Configured `ossec.conf` to collect the `Microsoft-Windows-Sysmon/Operational` channel

Details: [`04-windows-agent-sysmon`](04-windows-agent-sysmon/)

### 4. Mimikatz Detection
- Executed Mimikatz on the endpoint to generate telemetry
- Enabled full log archiving temporarily to inspect raw events
- Identified `originalFileName` as a reliable detection field
- Wrote a custom rule to alert on Mimikatz execution

Details: [`05-mimikatz-detection`](05-mimikatz-detection/)

### 5. Shuffle Automation
- Created a workflow with a webhook trigger
- Configured the Wazuh integration in `ossec.conf` to forward alerts to the webhook
- Extracted the SHA256 hash with a regex node
- Queried VirusTotal for hash reputation
- Created an alert in TheHive and sent an email notification to the analyst

Details: [`06-shuffle-automation`](06-shuffle-automation/)

## Detection Logic

Attackers commonly rename executables to evade file-name-based detection. This rule instead matches the `originalFileName` value from the PE metadata, which persists even when the file is renamed.

```xml
<group name="sysmon,">
  <rule id="100002" level="15">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.originalFileName" type="pcre2">(?i)mimikatz\.exe</field>
    <description>Mimikatz usage detected</description>
    <mitre>
      <id>T1003</id>
    </mitre>
  </rule>
</group>
```

![Mimikatz alert](05-mimikatz-detection/screenshots/mimikatz-alert.png)

## Extensions

The following work goes beyond the original series.

### 1. Renamed Mimikatz and LSASS Access Detection
- **Technique:** T1003.001, OS Credential Dumping: LSASS Memory
- **Scenario:** Renamed the binary to test whether the original rule still fires, then targeted behavior-based detection
- **Telemetry:** Sysmon Event ID 10 (ProcessAccess) with `TargetImage` = `lsass.exe`
- **Rule:** [`07-extensions/custom-rules/lsass-access.xml`](07-extensions/custom-rules/)
- **Result:** [Describe outcome and tuning]

### 2. Brute Force Attack
- **Technique:** T1110, Brute Force
- **Tool:** Hydra (from Kali Linux)
- **Rule:** [`07-extensions/custom-rules/brute-force.xml`](07-extensions/custom-rules/)
- **Result:** [Describe outcome]

### 3. Malicious PowerShell Execution
- **Technique:** T1059.001, Command and Scripting Interpreter: PowerShell
- **Tool:** Atomic Red Team
- **Rule:** [`07-extensions/custom-rules/powershell.xml`](07-extensions/custom-rules/)
- **Result:** [Describe outcome and false-positive tuning]

### 4. Automated Response
- Configured Wazuh Active Response to block source IPs after repeated failed logins
- Added IP reputation enrichment (AbuseIPDB) to the Shuffle workflow

## MITRE ATT&CK Coverage

| Tactic | Technique | ID | Rule ID | Source |
|---|---|---|---|---|
| Credential Access | LSASS Memory | T1003.001 | 100002 | MyDFIR series |
| Credential Access | LSASS Memory (behavioral) | T1003.001 | [100003] | Extension |
| Credential Access | Brute Force | T1110 | [100004] | Extension |
| Execution | PowerShell | T1059.001 | [100005] | Extension |

## Repository Structure

```
├── README.md
├── LICENSE
├── 01-architecture/
├── 02-wazuh-setup/
├── 03-thehive-setup/
├── 04-windows-agent-sysmon/
├── 05-mimikatz-detection/
├── 06-shuffle-automation/
└── 07-extensions/
    ├── attack-simulations/
    ├── custom-rules/
    └── screenshots/
```

All configuration files and exports are sanitized. IP addresses, API keys, and credentials have been removed.

## Challenges & Lessons Learned

| Challenge | Root Cause | Resolution |
|---|---|---|
| [Agent could not connect] | [Ports 1514/1515 blocked] | [Updated firewall rules] |
| [TheHive failed to start] | [Cassandra/Elasticsearch misconfiguration] | [Corrected configs and permissions] |
| [Shuffle received no alerts] | [Incorrect webhook URL] | [Fixed integration block] |

**Key takeaways**

- Detection quality depends heavily on telemetry quality and Sysmon configuration
- Behavior-based detections are more resilient than file-name-based ones
- Automation reduces triage time but requires tuning to avoid alert fatigue

## Future Improvements

- Add Linux endpoints and additional log sources
- Integrate MISP for threat intelligence
- Build detection coverage dashboards
- Validate rules automatically in a CI pipeline

## Credits

This project is based on the *SOC Automation Project* series by [MyDFIR](https://www.youtube.com/@MyDFIR). All content in [`07-extensions`](07-extensions/) is my own work.

**References**
- [Wazuh Documentation](https://documentation.wazuh.com)
- [TheHive Documentation](https://docs.strangebee.com)
- [Shuffle Documentation](https://shuffler.io/docs)
- [MITRE ATT&CK](https://attack.mitre.org)
- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config)
- [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team)

## Disclaimer

This project is for educational purposes only. All simulations were performed in an isolated environment that I own. Do not use these techniques against systems without authorization.

## Author

**[Xuan son]**
[LinkedIn](https://linkedin.com/in/your-profile) | [GitHub](https://github.com/your-username) | [Email](mailto:your@email.com)
