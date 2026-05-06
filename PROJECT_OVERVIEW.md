# 🔍 Azure–Splunk Hybrid Monitoring Lab

![Lab Status](https://img.shields.io/badge/status-active-brightgreen)
![Platform](https://img.shields.io/badge/platform-Azure-blue)
![SIEM](https://img.shields.io/badge/SIEM-Splunk-orange)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

---

## 📋 Table of Contents

- [Purpose](#purpose)
- [Architecture Summary](#architecture-summary)
- [Components](#components)
- [Data Flow](#data-flow)
- [Key Configurations](#key-configurations)
- [Learning Objectives](#learning-objectives)
- [SIEM & Log Analysis Skills Demonstrated](#siem--log-analysis-skills-demonstrated)
- [Getting Started](#getting-started)
- [Project Structure](#project-structure)

---

## Purpose

This lab simulates a **real-world hybrid monitoring environment** hosted on Microsoft Azure, using Splunk as the central Security Information and Event Management (SIEM) platform. The goal is to replicate the log collection, forwarding, indexing, and analysis pipeline found in enterprise security operations centers (SOCs).

By deploying a Windows endpoint alongside a dedicated Splunk server and connecting them via the Universal Forwarder, this lab provides hands-on experience with:

- End-to-end log pipeline architecture
- Windows event monitoring and threat detection
- Splunk Search Processing Language (SPL) for security investigations
- Cloud-hosted infrastructure management on Azure

This project serves as a **portfolio-grade demonstration** of SIEM engineering and log analysis proficiency applicable to SOC Analyst, Security Engineer, and Detection Engineering roles.

---

## Architecture Summary

```
┌─────────────────────────────────────────────────────────┐
│                     Microsoft Azure                      │
│                                                         │
│  ┌─────────────────────┐     ┌───────────────────────┐  │
│  │    Windows VM        │     │      Splunk VM         │  │
│  │  (Log Source)        │     │  (SIEM / Indexer)      │  │
│  │                     │     │                       │  │
│  │  - Windows Event Log │────▶│  - Splunk Enterprise  │  │
│  │  - Security Logs     │9997 │  - Indexer            │  │
│  │  - System Logs       │     │  - Search Head        │  │
│  │  - App Logs          │     │  - Web UI (:8000)     │  │
│  │                     │     │                       │  │
│  │  [UF Agent Running] │     │  [Receiving Enabled]  │  │
│  └─────────────────────┘     └───────────────────────┘  │
│                                                         │
│              Azure Virtual Network (VNet)               │
│         Network Security Groups (NSG) Applied           │
└─────────────────────────────────────────────────────────┘
```

Both virtual machines reside within the same **Azure Virtual Network (VNet)**, secured by **Network Security Groups (NSGs)** that restrict traffic to only the ports required for log forwarding and administration.

---

## Components

### 1. 🖥️ Windows Virtual Machine (Log Source)

| Property | Value |
|---|---|
| Role | Endpoint / Log Generator |
| OS | Windows Server 2019 / Windows 10 |
| Hosted On | Microsoft Azure |
| Key Service | Splunk Universal Forwarder |
| Logs Generated | Security, System, Application Event Logs |

The Windows VM acts as the **monitored endpoint**. It generates real Windows Event Logs (logon events, process creation, policy changes, etc.) and runs the Universal Forwarder agent to ship those logs to Splunk.

---

### 2. 🔎 Splunk Virtual Machine (SIEM Server)

| Property | Value |
|---|---|
| Role | SIEM / Log Aggregation Server |
| OS | Ubuntu 22.04 LTS |
| Hosted On | Microsoft Azure |
| Software | Splunk Enterprise (Free/Trial License) |
| Interfaces | Web UI (:8000), Receiver (:9997) |

The Splunk VM serves as the **centralized log aggregation and analysis platform**. It receives forwarded logs, indexes them for fast retrieval, and provides the Splunk Web interface for querying, dashboards, and alerting.

---

### 3. 📦 Splunk Universal Forwarder (UF)

| Property | Value |
|---|---|
| Role | Lightweight Log Shipping Agent |
| Installed On | Windows VM |
| Communication Port | TCP 9997 |
| Config File | `inputs.conf`, `outputs.conf` |

The Universal Forwarder is a **lightweight Splunk agent** installed on the Windows VM. It monitors configured log sources and forwards raw log data to the Splunk indexer over TCP port 9997 without performing any local indexing, minimizing resource overhead on the endpoint.

---

## Data Flow

```
Windows Event Logs          Universal Forwarder         Splunk Indexer
─────────────────           ───────────────────         ──────────────
  Security.evtx    ──────▶   Monitor inputs.conf  ──────▶  TCP :9997
  System.evtx               Forward via outputs.conf        │
  Application.evtx                                          ▼
                                                    Indexed in Splunk
                                                           │
                                                           ▼
                                                  Splunk Search Head
                                                  SPL Queries / Alerts
                                                  Dashboards / Reports
```

**Step-by-step flow:**

1. **Event Generation** — Windows OS writes Security, System, and Application events to the Windows Event Log (`.evtx` files).
2. **Log Monitoring** — The Universal Forwarder monitors these log channels as defined in `inputs.conf`.
3. **Forwarding** — The UF establishes a persistent TCP connection to the Splunk receiver on port 9997 and streams log data continuously.
4. **Indexing** — Splunk receives the raw events, parses them, applies source type mappings (`WinEventLog`), timestamps them, and writes them to the index.
5. **Search & Analysis** — Analysts use the Splunk Web UI and SPL to query, correlate, and visualize events for threat detection and investigation.

---

## Key Configurations

### Universal Forwarder — `inputs.conf`

```ini
[WinEventLog://Security]
index = main
disabled = false
start_from = oldest
evt_resolve_ad_obj = 1

[WinEventLog://System]
index = main
disabled = false

[WinEventLog://Application]
index = main
disabled = false
```

### Universal Forwarder — `outputs.conf`

```ini
[tcpout]
defaultGroup = splunk-indexer

[tcpout:splunk-indexer]
server = <SPLUNK_VM_PRIVATE_IP>:9997
```

### Splunk — Receiving Configuration

```ini
# Enabled via Splunk Web UI:
# Settings → Forwarding and Receiving → Configure Receiving → Port 9997
```

### Azure NSG Rules (Minimum Required)

| Priority | Name | Port | Protocol | Source | Destination | Action |
|---|---|---|---|---|---|---|
| 100 | Allow-RDP | 3389 | TCP | Your IP | Windows VM | Allow |
| 110 | Allow-SSH | 22 | TCP | Your IP | Splunk VM | Allow |
| 120 | Allow-SplunkWeb | 8000 | TCP | Your IP | Splunk VM | Allow |
| 130 | Allow-SplunkForwarder | 9997 | TCP | Windows VM | Splunk VM | Allow |

> ⚠️ **Security Note:** In production, RDP and SSH access should be gated behind Azure Bastion or a VPN. Exposing these ports directly to the internet is for lab purposes only.

---

## Learning Objectives

By completing and working with this lab, you will be able to:

- [ ] **Deploy and configure** cloud-hosted virtual machines on Microsoft Azure
- [ ] **Install and configure** the Splunk Universal Forwarder on a Windows endpoint
- [ ] **Enable and validate** log forwarding over TCP using `inputs.conf` and `outputs.conf`
- [ ] **Navigate the Splunk Web UI** to manage indexes, sourcetypes, and search interfaces
- [ ] **Write SPL queries** to search, filter, and aggregate Windows Event Log data
- [ ] **Identify and investigate** common Windows security events (Event IDs 4624, 4625, 4688, 4720, etc.)
- [ ] **Build Splunk dashboards** to visualize authentication activity and system health
- [ ] **Configure Azure NSGs** to enforce least-privilege network access
- [ ] **Document a security lab** to professional, portfolio-ready standards

---

## SIEM & Log Analysis Skills Demonstrated

This lab directly maps to skills evaluated in **SOC Analyst**, **Detection Engineer**, and **Security Operations** roles:

### 🔐 Windows Security Event Monitoring

| Event ID | Event Name | Relevance |
|---|---|---|
| 4624 | Successful Logon | Baseline activity; detect anomalous logins |
| 4625 | Failed Logon | Brute-force detection |
| 4648 | Logon with Explicit Credentials | Lateral movement indicator |
| 4688 | Process Creation | Malware execution, LOLBins |
| 4720 | User Account Created | Persistence detection |
| 4732 | Member Added to Security Group | Privilege escalation |
| 7045 | New Service Installed | Persistence mechanism |

### 📊 SPL Query Examples

```spl
# Failed logon attempts (brute force detection)
index=main source="WinEventLog:Security" EventCode=4625
| stats count by Account_Name, src_ip
| where count > 5
| sort -count

# New user account creation
index=main source="WinEventLog:Security" EventCode=4720
| table _time, Subject_Account_Name, Target_Account_Name, host

# Process creation audit
index=main source="WinEventLog:Security" EventCode=4688
| table _time, host, Account_Name, Process_Name, Creator_Process_Name

# Logon activity over time
index=main source="WinEventLog:Security" EventCode=4624
| timechart count by host
```

### 🧠 Core Competencies Evidenced

- **Log pipeline architecture** — end-to-end design from source to SIEM
- **Agent deployment** — installing and configuring endpoint telemetry agents
- **Data normalization** — sourcetype mapping and field extraction in Splunk
- **Threat hunting** — SPL-based investigation workflows
- **Cloud infrastructure** — Azure VM deployment, VNet design, and NSG policy
- **Documentation** — clear, reproducible technical documentation for audit and collaboration

---

## Getting Started

```bash
# Clone the repository
git clone https://github.com/<your-username>/azure-splunk-lab.git
cd azure-splunk-lab
```


## Project Structure

```
azure-splunk-lab/
├── PROJECT_OVERVIEW.md        # This file
├── docs/
│   ├── azure-vm-setup.md
│   ├── splunk-install.md
│   ├── universal-forwarder-setup.md
│   └── splunk-queries.md
├── configs/
│   ├── inputs.conf
│   └── outputs.conf
├── dashboards/
│   └── windows-security-overview.xml
└── screenshots/
    ├── splunk-dashboard.png
    └── forwarder-connected.png
```

---

*Built by [Sandy C.] | Azure • Splunk • SIEM • Log Analysis | May 2026*
