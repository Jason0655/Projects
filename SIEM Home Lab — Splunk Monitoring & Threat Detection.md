# 🛡️ SIEM Home Lab — Splunk Monitoring & Threat Detection

A self-built **Security Information and Event Management (SIEM)** home lab that ingests Windows security telemetry into Splunk Enterprise and detects live adversary activity simulated from a Kali Linux attack box.

![Status](https://img.shields.io/badge/status-complete-brightgreen)
![Focus](https://img.shields.io/badge/focus-Detection%20Engineering%20%7C%20Blue%20Team-blue)
![Stack](https://img.shields.io/badge/stack-Splunk%20%7C%20Sysmon%20%7C%20VirtualBox-orange)

## 📋 Overview

This project documents the end-to-end build of a functional SIEM environment: deploying Splunk Enterprise on Windows Server, forwarding Windows Event Logs and Sysmon telemetry into it, and validating detection coverage by launching real attacks (network scanning, brute-force auth) from an adversary VM and confirming they surface as alerts in Splunk.

**Goal:** Develop practical detection-engineering skills — log ingestion, index/sourcetype configuration, and writing SPL (Search Processing Language) queries to catch reconnaissance and credential-attack techniques.

## 🏗️ Lab Architecture

| Component | Role | IP Address |
|---|---|---|
| **Windows Server 2022** | Splunk Enterprise (Indexer, Search Head, Deployment Manager) + Sysmon endpoint | `192.168.1.50` (static) |
| **Kali Linux** | Adversary simulation — scanning, brute-forcing, exploitation | `192.168.1.60` (static) |
| **Hypervisor** | Oracle VM VirtualBox (Host-Only / Bridged network) | — |

> Both VMs use static IPs on an isolated VirtualBox network segment to keep the ingest pipeline stable and predictable.

## 🔧 Build Steps

### 1. Network Configuration
Static IPs assigned on both hosts:
- **Windows:** via Adapter Settings → Internet Protocol Version 4, validated with `ipconfig /all`
- **Kali:**
  ```bash
  sudo ip addr add 192.168.1.60/24 dev eth1
  sudo ip link set eth1 up
  ip addr show eth1
  ```

### 2. Install & Configure Splunk Enterprise
1. Install the Splunk Enterprise MSI on Windows Server
2. Access the web console at `http://localhost:8000` (or `http://192.168.1.50:8000`)
3. **Settings → Forwarding and receiving → Configure receiving** — open port `9997` for forwarder traffic

### 3. Deploy Sysmon (Advanced Endpoint Telemetry)
Standard Windows Event Logs miss detailed process-execution data, so [Sysmon](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon) is installed with the community-maintained [SwiftOnSecurity config](https://github.com/SwiftOnSecurity/sysmon-config):

```
sysmon64.exe -i sysmonconfig.xml
```

### 4. Install the Splunk Universal Forwarder
Configured to forward both Windows Security logs and Sysmon operational logs via `inputs.conf`:

```ini
[WinEventLog://Security]
disabled = 0
index = main

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = false
renderXml = true
index = main
sourcetype = XmlWinEventLog:Sysmon
```

Restart the service to apply:
```
net stop SplunkForwarder && net start SplunkForwarder
```

Verify the connection:
```
./splunk list forward-server
```

## 🎯 Detection Scenarios & Results

### Scenario A — Active Reconnaissance (Nmap Sweep)
- **Attack:** Nmap target sweep launched from Kali against the Windows Server
- **Detection:** SPL query in Splunk identifies spikes in firewall permit validations across restricted ports
- **Result:** ✅ Reconnaissance activity surfaced in Search & Reporting

### Scenario B — Brute-Force Authentication (RDP)
- **Attack:**
  ```bash
  hydra -l Administrator -P /usr/share/wordlists/rockyou.txt rdp://192.168.1.50
  ```
- **Detection:** SPL query isolates high-volume **Event ID 4625** ("An account failed to log on") within a short time window
- **Result:** ✅ Brute-force pattern clearly visible as an event-count anomaly

## 🛠️ Troubleshooting Framework

### Phase 1 — Local OS Telemetry
| Issue | Cause | Fix |
|---|---|---|
| Inbound packets silently dropped, Nmap results unreliable | Windows Defender Firewall / real-time protection blocking scans | `Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled False`<br>`Set-MpPreference -DisableRealtimeMonitoring $true` *(lab only — re-enable after testing)* |
| No Event ID 5156 (allowed connections) generated | Windows doesn't log allowed connections by default | `auditpol /set /subcategory:"Filtering Platform Connection" /success:enable /failure:enable` then `gpupdate /force` |

### Phase 2 — Splunk Transport & Indexing
| Issue | Cause | Fix |
|---|---|---|
| Forwarder inactive / errors listing forward-servers | Receiving port not enabled | Enable port `9997` under Settings → Forwarding and receiving → Receive data; verify with `Get-NetTCPConnection -LocalPort 9997` |
| `splunkd` won't restart cleanly | Hung process during connection retry | `Stop-Process -Name splunkd -Force` then `Start-Service -Name SplunkForwarder` |
| Logs confirmed on the wire but zero search results | Forwarder routing to a non-existent `windows` index | Explicitly set `index = main` in `inputs.conf` for the relevant stanza |

## 📚 References

- [Splunk Enterprise Download](https://www.splunk.com/en_us/download/splunk-enterprise.html)
- [Microsoft Sysinternals Suite (Sysmon)](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [Splunk Deployment & Security Architecture Docs](https://docs.splunk.com/Documentation/Splunk/latest/Installation/Whatinthismanual)
- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config)

## ⚠️ Disclaimer

Built entirely in an isolated home-lab network (VirtualBox host-only/bridged segment). Credentials shown in the original lab notes have been redacted here — **never commit real credentials to a public repository.** All offensive techniques were run only against lab-owned infrastructure.

---

**Author:** Oluwatoni Oderinlo
