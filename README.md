# Splunk SIEM & Log Analysis Lab

This lab walks through deploying Splunk Enterprise on an Azure Ubuntu VM, configuring network access, and preparing the environment to receive Windows event logs from a separate Active Directory lab VM.

---

## 📌 Prerequisites

Before starting, ensure you have:

- Basic understanding of Azure resources and networking
- An active Azure account (Free Tier is sufficient)
- An existing Windows VM from your **Active Directory Lab** in Azure

---

## 🥇 Step 1 — Download Splunk Enterprise (Free)

[![Splunklab](https://github.com/user-attachments/assets/ea2c63fc-1d34-4589-8add-c40bf438a1ec)](https://www.loom.com/share/5e0b6fe077ca47f3ad135645fa0c7dbe)

Splunk Enterprise offers a **60‑day full trial**, then automatically converts to the **free license** (500MB/day indexing — perfect for labs).

1. Visit: https://splunk.com/en_us/download/splunk-enterprise.html  
2. Splunk requires a business email.  
   - If you prefer not to use your work email, generate a temporary one at: https://temp-mail.org  
   - Keep the tab open — Splunk will send a confirmation link and save the credentials.
3. Copy the **Linux download link** for Splunk Enterprise (you’ll use it on the Ubuntu VM).

---

## 🥈 Step 2 — Deploy the Azure VM for the Splunk Server

1. Go to https://azure.microsoft.com/free and create an account (if needed).
2. Sign in to the Azure Portal: https://portal.azure.com.
3. Search for **Virtual Machines** → **Create**.
4. Use the following configuration:

| Setting | Value |
|--------|-------|
| Resource Group | `RG-Splunk-Lab` |
| VM Name | `VM-Splunk-Lab` |
| Region | East US |
| Image | Ubuntu 22.04 LTS (Free Tier eligible) |
| Size | Standard_B2s (2 vCPU, 4GB RAM minimum) |
| Authentication | SSH public key -  Provide a Key Pair name: SplunkKey |
| Public inbound ports | Allow SSH (22) |
| OS Disk | Standard SSD |

5. Click **Review + Create**, then **Create**.
6. Download and save the SSH key — you’ll need it to connect to the VM.

---

### **Update the Network Security Group (NSG)**

Splunk requires specific ports to be open so you can access the UI and receive logs.

### **Allow inbound traffic on port 8000 (Splunk Web UI)**

1. Go to **RG-Splunk-Lab** → open the VM’s **Network Security Group**.
2. Select **Inbound security rules** → **Add**.
3. Enter:

| Setting | Value |
|--------|-------|
| Source | Any *(Lab only — in production restrict this)* |
| Source Port | * |
| Destination | Any |
| Service | Custom |
| Destination Port | **8000** |
| Protocol | Any |
| Action | Allow |
| Priority | 310 |
| Name | `Allow-SplunkWebUI` |

Click **Add**.

---

### **Allow inbound traffic on port 9997 (Splunk Forwarder Input)**

1. In the same NSG → **Inbound security rules** → **Add**.
2. Enter:

| Setting | Value |
|--------|-------|
| Source | Any *(Lab only)* |
| Source Port | * |
| Destination | Any |
| Service | Custom |
| Destination Port | **9997** |
| Protocol | Any |
| Action | Allow |
| Priority | 320 |
| Name | `Allow-SplunkForwarder-WindowsVM` |

Click **Add**.

---

### ** Create VNET Peering Between the Two VMs **

This allows your Splunk VM and your AD Lab VM to communicate privately.

1. Go to **RG-Splunk-Lab** → open the **Virtual Network**.
2. Under **Settings**, select **Peerings** → **Add**.
3. Fill in:

### **Remote Virtual Network Summary**
| Setting | Value |
|--------|-------|
| Peering link name | `Splunk-VNET-to-AD-VNET` |
| Peering type | VNET |
| Subscription | Your Azure subscription |
| Virtual Network | `VM-AD-DC01-vnet` |
| Allow access | ✔ Allow VM-AD-DC01-vnet to access Splunk-Lab-vnet |

### **Local Virtual Network Summary**
| Setting | Value |
|--------|-------|
| Peering link name | `AD-VNET-to-Splunk-VNET` |
| Allow access | ✔ Allow Splunk-Lab-vnet to access VM-AD-DC01-vnet |

Click **Add**.

---

## 🥉 Step 3  — Download & Install Splunk Enterprise on the Linux VM

This section covers connecting to your Ubuntu VM using SSH, converting your Azure key for PuTTY, and installing Splunk Enterprise.

---

## 🔐 Connect to the Linux VM via SSH (Using PuTTY)

### **Install PuTTY (Windows)**

1. Go to https://putty.org → click **Download PuTTY**
2. Under *Package files*, download the **64‑bit Windows Installer (.msi)**
3. Run the installer and accept all defaults
4. Launch **PuTTY** from the Start Menu

---

### **Convert Azure PEM Key to PPK (Required for PuTTY)**

1. Open **PuTTYgen** (installed with PuTTY)
2. Click File, then select **Load private key** → select your `.pem` file  
   *(Change file type dropdown to “All Files” if needed)*
3. Once loaded, click **Save private key**
4. Choose a location and save the `.ppk` file
5. Your key is now ready for SSH authentication

---

### **SSH into the VM Using PuTTY**

1. Open **PuTTY**
2. In **Hostname**, enter your VM’s **Public IP address**
3. In the left menu:  
   `Connection → SSH → Auth → Credentials`  
   → Browse for your `.ppk` file under **Private key file for authentication**
4. Click **Open** to start the session
5. When prompted, enter your **VM username**

---

## 📥 Download & Install Splunk Enterprise on Ubuntu

### **Download Splunk**
Run the following command inside your SSH session:

```bash
wget -O splunk-10.2.3-4d61cf8a5c0c.x86_64.rpm https://download.splunk.com/products/splunk/releases/10.2.3/linux/splunk-10.2.3-4d61cf8a5c0c.x86_64.rpm
```

### **Install Splunk**

```bash
sudo dpkg -i splunk-10.2.2-linux-amd64.deb
```

### **Start Splunk & Accept License**

You will be prompted to create admin credentials:

```bash
sudo /opt/splunk/bin/splunk start --accept-license
```

### **Enable Splunk to Start on Boot**
```bash
sudo /opt/splunk/bin/splunk enable boot-start
```

### **Access the Splunk Web UI**

Open your browser and go to:
```bash
http://<YOUR_VM_PUBLIC_IP>:8000
```

---

## 🧩 Step 4 — Configure Data Inputs in Splunk

Splunk becomes useful only when it begins receiving data.  
In this step, you will configure Splunk Enterprise to receive logs from your **Windows Server / Active Directory VM** using the **Splunk Universal Forwarder**.

---

## 🔌 Enable Receiving in Splunk

Log in to Splunk Web using the admin credentials you created during installation.

1. Go to **Settings → Forwarding and Receiving**
2. Select **Configure Receiving → New Receiving Port**
3. Enter: **9997** → Save  
   *(This is the default port for Splunk Forwarder traffic.)*
4. Go to **Settings → Indexes → New Index**
5. Create an index named: **windows_logs**

---

## 📦 Install the Splunk Universal Forwarder on the Windows Server (AD VM)

Remote into your Windows Server VM from Lab 1.

1. On the Windows Server, open a browser and go to:  
   https://splunk.com/en_us/download/universal-forwarder.html  
   → Sign in using the Splunk account created earlier.
2. Download the **Windows 64‑bit Universal Forwarder installer**
3. Run the installer:
   - Under **Use this UniversalForwarder with**, select:  
     **An on-premise Splunk Enterprise instance**
4. Create a username and password for the forwarder
5. When prompted for a **Deployment Server**, leave it blank → Next
6. When prompted for a **Receiving Indexer**, enter:
   - **Splunk VM Private IP**
   - **Port 9997**
7. Complete the installation with default settings

---

## 📝 Configure `inputs.conf` to Specify Which Logs to Forward

The `inputs.conf` file tells the forwarder exactly which Windows Event Logs to collect.

### **1. Install Visual Studio Code on the VM Splunk Server**  
Download from: https://code.visualstudio.com/download

### **2. Navigate to the correct folder**

Go to:

C:\Program Files\SplunkUniversalForwarder\etc\system\local\

If the **local** folder does not exist, create it.


### **3. Open the folder in VS Code**

In File Explorer, type: **code .** and it is going to open the directory  C:\Program Files\SplunkUniversalForwarder\etc\system\local\ on VisualStudio Code


### **4. Create a new file named `inputs.conf`**

Paste the following configuration:

```ini
# File location:
# C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf

[WinEventLog://Security]
disabled = 0
start_from = oldest
current_only = 0
evt_resolve_ad_obj = 1
index = windows_logs

[WinEventLog://System]
disabled = 0
index = windows_logs

[WinEventLog://Application]
disabled = 0
index = windows_logs
```

Save the changes 

### **5. Restart the Splunk Forwarder**

Open a PowerShell window as admin and restart the forwarder to apply the changes with the following command:

```powershell
Restart-Service SplunkForwarder
```

---


## 📝 Generate Windows Event Logs (Traffic Simulation)

To test ingestion, you will generate authentication events, service events, application warnings, and account lockouts.

Run the following script in PowerShell ISE (as Administrator)

```powershell
# Lab 3 Log Generator - Run as Administrator on the Windows VM
# Output saved to C:\lab3-log-output.txt

$logFile = 'C:\lab3-log-output.txt'
$timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'

function Log($message, $color = 'White') {
    Write-Host $message -ForegroundColor $color
    Add-Content -Path $logFile -Value "[$timestamp] $message"
}

# Clear previous output
if (Test-Path $logFile) { Remove-Item $logFile }
Add-Content -Path $logFile -Value "Lab 3 Log Generator - Run started at $timestamp"
Add-Content -Path $logFile -Value '==============================================='

Log 'Starting log generation...' -Color Green

# Create test user
$testUser = 'labtest.user'
$testPass = ConvertTo-SecureString 'TempPass123!' -AsPlainText -Force
New-LocalUser -Name $testUser -Password $testPass -Description 'Splunk lab test account' -ErrorAction SilentlyContinue
Log "Created test user: $testUser" -Color Gray

# Failed logins (4625)
Log 'Generating failed login attempts (Event ID 4625)...' -Color Yellow
$wrongPass = ConvertTo-SecureString 'WrongPassword!' -AsPlainText -Force
1..15 | ForEach-Object {
    $cred = New-Object System.Management.Automation.PSCredential($testUser, $wrongPass)
    Start-Process -FilePath cmd.exe -Credential $cred -ArgumentList '/c exit' -ErrorAction SilentlyContinue
    Start-Sleep -Milliseconds 500
}
Log 'Generated 15 failed login attempts (Event ID 4625)' -Color Gray

# Successful login (4624)
Log 'Generating successful login (Event 4624)...' -Color Yellow
$correctPass = ConvertTo-SecureString 'TempPass123!' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential($testUser, $correctPass)
Start-Process -FilePath cmd.exe -Credential $cred -ArgumentList '/c whoami' -Wait -ErrorAction SilentlyContinue
Log 'Generated successful login (Event ID 4624)' -Color Gray

# Service events (7036)
Log 'Generating service events (7036)...' -Color Yellow
$services = @('Spooler','Schedule','Netlogon')
$services | ForEach-Object {
    Start-Service -Name $_ -Force -ErrorAction SilentlyContinue
    Start-Sleep -Seconds 2
    Stop-Service -Name $_ -ErrorAction SilentlyContinue
    Start-Sleep -Seconds 1
    Log "Stopped and restarted service: $_" Gray
}

# Application warnings (1001)
Log 'Generating application log events...' Yellow
$eventSource = "SplunkLabTest"
if (-not [System.Diagnostics.EventLog]::SourceExists($eventSource)) {
    New-EventLog -LogName Application -Source $eventSource -ErrorAction SilentlyContinue
}

1..5 | ForEach-Object {
    Write-EventLog -LogName Application -Source $eventSource -EventId 1001 `
    -EntryType Warning -Message "Splunk lab test event 'application warning'"
    Start-Sleep -Milliseconds 300
}
Log 'Generated 5 application log entries (Event ID 1001)' Gray

# Account lockout (4740)
Log 'Generating account lockout event (4740)...' Yellow
$badCred = ConvertTo-SecureString 'BadPass!' -AsPlainText -Force
1..2 | ForEach-Object {
    $cred = New-Object System.Management.Automation.PSCredential($testUser, $badCred)
    Start-Process -FilePath 'cmd.exe' -Credential $cred -ArgumentList '/c exit' -ErrorAction SilentlyContinue
    Start-Sleep -Milliseconds 200
}
Log 'Account lockout triggered (Event ID 4740)' Gray

# Cleanup
Start-Sleep -Seconds 1
Remove-LocalUser -Name $testUser -ErrorAction SilentlyContinue
Log "Removed test user: $testUser" Gray

# Restart forwarder
Log 'Restarting forwarder to ship events to Splunk...' Yellow
Restart-Service SplunkForwarder -ErrorAction SilentlyContinue
Log 'SplunkForwarder restarted' Gray

$endTime = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
Add-Content -Path $logFile -Value '========================================================'
Add-Content -Path $logFile -Value "Run completed at $endTime"
Add-Content -Path $logFile -Value "Wait 60 seconds then run: 'index=windows_logs | head 100'"

```

---

Click Run Script in PowerShell ISE

Wait ~2 minutes for Splunk to ingest the events

---


## 🧠 Step 5 — Essential SPL Searches

All searches are run in the **Search & Reporting** app in Splunk.  
Use the **time picker** (top right) to select the time range before running each query.

---

### ✔️ Confirm Data Is Flowing

```spl
index=windows_logs | head 100
```

What this tells you::

index=windows_logs — searches only in the index you created

| head 100 — returns the first 100 events

If you see results, your forwarder is working.
If not, verify the SplunkForwarder service is running on the Windows VM.

---

### 🔐 Find Failed Login Attempts (EventCode 4625)

```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4625
| stats count by Account_Name, Workstation_Name
| sort -count
```

What this tells you:
 - EventCode=4625 — failed logon attempts
 - stats count by ... — shows which accounts and machines generated failures
 - High counts (5+ in a short time) may indicate brute force attempts

---

###  🔓 Find Successful Logins (EventCode 4624)

```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4624
| stats count by Account_Name, Logon_Type
| sort -count
```

Logon Types to know:

 2 — Interactive (local keyboard)
 
 3 — Network (file shares, SMB)
 
 5 — Service account
 
 10 — Remote interactive (RDP)

---

###  🔒 Find Account Lockouts (EventCode 4740)

```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4740
| table _time, Account_Name, Caller_Computer_Name
| sort -_time
```

Why this matters:
 - Shows when an account was locked
 - Caller_Computer_Name identifies where the failed attempts originated
 - Multiple lockouts = possible password spray or brute force

---


###  📊 Top 10 Failed Login Usernames (Threat Hunting)
```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4625 earliest=-24h
| stats count as failures by Account_Name
| sort -failures
| head 10
```

Use cases:
 - Identify targeted accounts
 - Spot enumeration attempts
 - Unknown usernames = attacker probing AD

---

🌙 Detect After‑Hours Logins
```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4624
| eval hour=strftime(_time, "%H")
| where hour < 7 OR hour > 19
| table _time, Account_Name, Workstation_Name, Logon_Type
| sort -_time

```

Interpretation:
 - After‑hours service account logins (Type 5) are normal
 - After‑hours interactive logins (Type 2 or 10) may require investigation

---

## 📊 Step 6 — Build a Security Dashboard

Dashboards provide a real‑time view of your security posture.  
You will create a **Windows Security Overview** dashboard using the searches from Step 5.

---

### 🛠️ Create the Dashboard

1. In Splunk, go to **Dashboards**
2. Click **Create New Dashboard**
3. Name it: **Windows Security Overview**
4. Click **Create Dashboard**
5. Click **Add Panel** for each of the panels below
6. For each panel:
   - Choose **New Search**
   - Paste the SPL query
   - Select a visualization type
   - Save

---

### 📌 Recommended Panels

| Panel Name | SPL Search | Visualization |
|-----------|------------|---------------|
| **Failed Logins — Last 24h** | `EventCode=4625` with `stats count by Account_Name` | Bar Chart |
| **Account Lockouts — Last 7d** | `EventCode=4740` with table output | Events List |
| **Login Activity Over Time** | `EventCode=4624` with `timechart count` | Line Chart |
| **Top Source IPs — After Hours** | After‑hours search with `stats count by Workstation_Name` | Column Chart |

---


## 🚨 Step 7 — Create an Automated Alert

Alerts allow Splunk to notify you when suspicious activity occurs — just like a real SOC workflow.

---

### ✔️ First, test the search manually

```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4625
| stats count as failures by Account_Name
| where failures > 10
```

What this detects:
 - Any account with more than 10 failed logins
 - In real environments, this is a strong indicator of brute force attacks

---
### 🛎️ Create the Alert

Click Save As → Alert

| Setting | Value |
|--------|-------|
| Name | Potential Brute Force — High Failure Count |
| Alert Type | Scheduled |
| Run every | 15 minutes |
| Trigger Condition | Number of Results > 0 |
| Trigger Actions | Add to Triggered Alerts |

Click Save

Your Splunk instance will now automatically detect brute force attempts.




