## Components

| Component | Function |
|---|---|
| Windows 10 Client | Generates endpoint events (Sysmon) and runs the Wazuh agent |
| Wazuh Manager | Analyzes events, applies detection rules, generates alerts |
| Shuffle | SOAR platform that orchestrates enrichment and response |
| VirusTotal | Provides file hash reputation |
| TheHive | Case management for analyst investigation |
| SOC Analyst | Receives email notification and decides on response |

## Data Flow

1. Windows client generates events, which the Wazuh agent forwards to the manager.
2. Wazuh evaluates events against rules and raises an alert on a match.
3. The alert is sent to Shuffle via webhook.
4. Shuffle extracts the SHA256 hash and queries VirusTotal.
5. Shuffle creates an alert in TheHive with the enrichment data.
6. Shuffle emails the analyst with details and a suggested next step.

