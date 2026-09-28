# Wazuh All-in-One Installation

> **Note:** Root privileges are required to run the commands below. Use `sudo` or switch to root with `sudo su -`.

## Overview

[Wazuh](https://wazuh.com) is a free, open-source security platform that combines **SIEM** (Security Information and Event Management) and **XDR** (Extended Detection and Response) capabilities. It collects and analyzes security data from endpoints, servers, cloud workloads, and network devices, then raises alerts when suspicious activity is detected.

**Key capabilities**

- Log data collection and analysis
- Intrusion and threat detection with a rule-based engine
- File integrity monitoring (FIM)
- Vulnerability detection
- Security configuration assessment (SCA)
- Active response to threats
- MITRE ATT&CK mapping and regulatory compliance support

### Architecture

| Component | Role |
|---|---|
| **Wazuh Agent** | Installed on monitored endpoints. Collects events and forwards them to the server. |
| **Wazuh Server (Manager)** | Receives agent data, decodes events, and applies detection rules to generate alerts. |
| **Wazuh Indexer** | Stores and indexes alerts and events, making them searchable. |
| **Wazuh Dashboard** | Web interface for visualization, threat hunting, and management. |
| **Filebeat** | Ships alerts and archives from the server to the indexer. |

In an **All-in-One** deployment, the server, indexer, and dashboard run on a single machine. This is suitable for labs and small environments.

## Environment

| Item | Value |
|---|---|
| Hypervisor | VMware Workstation |
| OS | Ubuntu 24.04.5 LTS |
| Hostname | `wazuh-server` |
| IP address | `192.168.29.131` |
| Resources | 2 vCPU / 4 GB RAM / 100 GB disk |

## Ports

| Port | Protocol | Purpose |
|---|---|---|
| 1514 | TCP | Agent communication |
| 1515 | TCP | Agent enrollment |
| 55000 | TCP | Wazuh server API |
| 9200 | TCP | Wazuh indexer API |
| 443 | TCP | Wazuh dashboard |

## Installation

### 1. Log in and connect via SSH

Log in on the VM console, then connect from the Windows host for easier copy/paste:

```bash
ssh wazuh-server@192.168.29.131
```


### 2. Update the system

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

### 3. Run the all-in-one installer

```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

The `-a` flag installs the Wazuh indexer, server, and dashboard on the same host. When the installation finishes, the script prints the `admin` credentials for the dashboard.



### 4. Retrieve the generated credentials

The installer creates an archive named `wazuh-install-files.tar` in the current directory. Extract it with elevated privileges:

```bash
sudo tar -xf wazuh-install-files.tar
sudo su -
cd /home/<user>/wazuh-install-files/
cat wazuh-passwords.txt
```

Without root, extraction and `cd` fail with `Permission denied`.


> **Security:** Never commit real credentials. Blur or remove passwords from screenshots before publishing.

### 5. Access the dashboard

Open `https://192.168.29.131` in a browser (accept the self-signed certificate warning) and log in as `admin` with the password from `wazuh-passwords.txt`.


## Post-Installation Configuration

By default, Wazuh only stores events that trigger a rule. To inspect **all** events (needed later for building custom detection rules), enable full log archiving.

### 6. Explore the Wazuh directory

Wazuh files live in `/var/ossec`, which is only accessible by root:

```bash
sudo su -
ls -la /var/ossec
cd /var/ossec/etc
ls
```


### 7. Enable log archiving on the manager

Edit the manager configuration:

```bash
nano /var/ossec/etc/ossec.conf
```

In the `<global>` section, set:

```xml
<logall>yes</logall>
<logall_json>yes</logall_json>
```


Save with `Ctrl+X`, `Y`, `Enter`, then restart the manager:

```bash
systemctl restart wazuh-manager.service
```


### 8. Enable archives in Filebeat

Filebeat must forward the archived events to the indexer:

```bash
cd /etc/filebeat/
nano filebeat.yml
```

Under `filebeat.modules`, set `archives` to enabled:

```yaml
filebeat.modules:
  - module: wazuh
    alerts:
      enabled: true
    archives:
      enabled: true
```

Restart Filebeat:

```bash
systemctl restart filebeat.service
```


## Verification

Check that all services are running:

```bash
systemctl status wazuh-manager
systemctl status wazuh-indexer
systemctl status wazuh-dashboard
systemctl status filebeat
```

Archived events are written locally to `/var/ossec/logs/archives/archives.json`. To search them in the dashboard, create a `wazuh-archives-*` index pattern under **Dashboard management**.

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| `Permission denied` when extracting the tar file or opening `/var/ossec` | Files are owned by root | Use `sudo` or switch to root with `sudo su -` |
| [Add your own issue] | [Cause] | [Fix] |

## Notes

- `logall` stores every event and can consume disk space quickly. Disable it when it is no longer needed.
- Use the installer version that matches your target Wazuh release.
- Change the default credentials for any environment beyond a personal lab.

## Result

The Wazuh server (manager, indexer, dashboard) is running with full event archiving enabled and the dashboard is accessible. The server is ready for agent enrollment.

## References

- [Wazuh Documentation](https://documentation.wazuh.com)
- [Wazuh Installation Guide](https://documentation.wazuh.com/current/installation-guide/index.html)
