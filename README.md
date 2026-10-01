# SOC Home Lab — Splunk SIEM, Windows Server & Kali Linux

A self-built, end-to-end SOC detection lab simulating a real log-forwarding and threat-detection pipeline: an attacker machine, a monitored target, and a centralized SIEM — built and troubleshot entirely from scratch.

## Overview

This lab was built to go beyond SOC theory and actually experience the full workflow a SOC Analyst deals with daily: configuring log forwarding, keeping a SIEM indexer healthy, simulating realistic attacker behavior, and writing detections against the resulting telemetry.

## Architecture

![Architecture Diagram](screenshots/architecture-diagram.png)

| Role | OS | Purpose |
|---|---|---|
| **Indexer** | Ubuntu 26.04 LTS | Runs Splunk Enterprise — stores, indexes, and searches all incoming logs |
| **Target** | Windows Server | Runs a Splunk Universal Forwarder shipping Security Event Logs to the indexer; also the attack target |
| **Attacker** | Kali Linux | Simulates reconnaissance and authentication attacks against the target |

> IP addresses in this README are shown as generic placeholders (`<indexer-ip>`, `<target-ip>`, `<attacker-ip>`) — replace with your own lab's addressing if you rebuild this.

## Setup

### 1. Indexer (Ubuntu)
- Installed Splunk Enterprise
- Configured to receive forwarded data on port `9997`

### 2. Forwarder (Windows Server)
- Installed the Splunk Universal Forwarder
- `outputs.conf` → pointed at `<indexer-ip>:9997`
- `inputs.conf` → set to monitor the Windows Security Event Log channel

See [`configs/`](configs/) for sanitized example config files.

### 3. Attack Simulation (Kali)
```bash
# Service/version enumeration
nmap -sV <target-ip>

# Simulated credential-guessing attempt
smbclient -L //<target-ip> -U fakeusers
```

## Problems Faced & Fixes

### 1. Splunk stopped indexing — disk space exhausted

Splunk's dashboard began throwing:
> "The index processor has paused data flow. Current free disk space... has fallen below the minimum of 5000MB."

The indexer had effectively gone blind — a real failure mode SOC teams hit when the underlying infrastructure runs out of resources.

**Diagnosis:**
```bash
df -h /
```
Confirmed the root partition had under 5GB free. Short-term relief (`apt clean`, `journalctl --vacuum-size=100M`, clearing Splunk's dispatch directory) helped briefly, but the VM disk was fundamentally undersized for a SIEM workload.

**Permanent fix — resized the virtual disk and extended the filesystem live:**
```bash
# After expanding the virtual disk to 60GB from the hypervisor
lsblk                                                          # confirm new raw space
sudo growpart /dev/sda 3                                       # grow the partition
sudo pvresize /dev/sda3                                        # resize LVM physical volume
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv   # extend logical volume
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv                # grow the filesystem
df -h /                                                         # confirm space recovered
```
Splunk resumed indexing automatically once free space cleared the threshold.

**Takeaway:** SIEM reliability is as much a storage/infrastructure problem as a detection-engineering one.

### 2. Attack simulation generated zero logs

After the disk fix, re-running the SMB attempt from Kali produced nothing in Splunk. The actual error was `NT_STATUS_IO_TIMEOUT` — not an authentication failure.

That distinction matters: a **timeout** means the connection never reached the target service at all, so Windows never generates an event. A real **failed login** would trigger Event ID `4625` regardless of Splunk.

**Diagnosis:**
```bash
nmap -p 445 <target-ip>
```
Confirmed port 445 (SMB) wasn't reachable.

**Fix (on Windows Server):**
```powershell
Get-Service -Name LanmanServer    # confirm SMB service is running

netsh advfirewall firewall add rule name="Allow SMB" dir=in action=allow protocol=TCP localport=445
```

Re-ran the SMB attempt from Kali — it reached Windows, failed authentication as expected, and logged Event ID `4625`.

## Validating the Pipeline End-to-End

```spl
index=* sourcetype=WinEventLog:Security EventCode=4625
```

Failed-login events appeared immediately, confirming the full chain: attack on Kali → auth failure on Windows → logged locally → forwarded → indexed → searchable.

![Event 4625 Search Results](screenshots/event-4625-search.png)

## Detection Engineering

Raw events aren't a detection. Built a correlation search to flag brute-force patterns instead of single failed attempts:

```spl
index=* sourcetype=WinEventLog:Security EventCode=4625
| stats count by src, user
| where count > 5
```

See [`searches/`](searches/) for saved `.spl` queries.

## Next Steps

- [ ] Convert the brute-force search into a scheduled Splunk alert
- [ ] Build a dashboard panel for failed-login trends and top source IPs
- [ ] Map detections to MITRE ATT&CK (e.g., T1110 – Brute Force)
- [ ] Add Sysmon on the Windows Server for deeper process/network telemetry

## What I Learned

The hardest parts of this lab weren't detection logic — they were disk space and a blocked firewall port. Troubleshooting them required real understanding of Linux storage (partitions, LVM, filesystems) and Windows service/firewall management, and reading a symptom correctly (timeout vs. auth failure) instead of guessing.

AI assistance was used throughout — particularly for unfamiliar Linux/LVM and Windows networking commands — with each step verified before running rather than copy-pasted blindly.

## Tools & Skills

`Splunk` `SIEM` `Windows Event Logs` `Nmap` `SMB Enumeration` `Linux Administration (LVM)` `Windows Server` `Firewall Configuration` `Detection Engineering` `Incident Detection`

---

**Author:** Rachit Tyagi · [LinkedIn](https://www.linkedin.com/in/rachit-tyagi-7b840322a)
