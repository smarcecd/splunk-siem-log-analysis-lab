# Architectural Diagram — Splunk SIEM & Log Analysis Lab

This document illustrates how logs flow from the Windows Active Directory VM into Splunk Enterprise running on an Ubuntu VM in Azure.

---

## 🏗️ High-Level Architecture




---

## 🔍 Component Breakdown

### **Azure Subscription**
Hosts both VMs:
- Windows Server (AD + log source)
- Ubuntu VM (Splunk Enterprise)

---

### **Virtual Networks & Peering**
- **AD-Lab-VNET** → Windows Server / AD VM  
- **Splunk-Lab-VNET** → Ubuntu Splunk VM  
- **VNET Peering** enables private communication between them.

---

### **Windows Server / AD VM**
- Generates Windows Event Logs:
  - Security  
  - System  
  - Application  
- Runs **Splunk Universal Forwarder**
- Sends logs to Splunk on port **9997**

---

### **Splunk Enterprise (Ubuntu VM)**
- Receives logs on **9997**
- Indexes them into **windows_logs**
- Provides:
  - Search & Reporting
  - Dashboards
  - Alerts
- Web UI available on **port 8000**

---

### **Analyst / Student Workstation**
- Connects to Splunk Web UI  
- Runs SPL queries  
- Builds dashboards  
- Reviews alerts  

---

## 🔄 Log Flow Summary

1. Windows Server generates event logs.  
2. Splunk Universal Forwarder sends logs to Splunk Enterprise (port 9997).  
3. Splunk indexes logs into `windows_logs`.  
4. Analyst accesses Splunk Web UI (port 8000).  
5. Analyst performs searches, dashboards, and alerting.  

---

