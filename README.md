# Comprehensive Guide: Monitoring Windows 10 VM Logs with Splunk Enterprise and Sysmon

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Architecture Overview](#architecture-overview)
3. [Splunk Enterprise Configuration](#splunk-enterprise-configuration)
4. [Windows 10 VM Setup](#windows-10-vm-setup)
5. [Sysmon Installation](#sysmon-installation)
6. [Universal Forwarder Configuration](#universal-forwarder-configuration)
7. [Data Inputs Configuration](#data-inputs-configuration)
8. [Splunk App Setup](#splunk-app-setup)
9. [Verification and Testing](#verification-and-testing)
10. [Troubleshooting](#troubleshooting)

## Prerequisites

### Software Requirements
- Splunk Enterprise installed on Windows host
- Windows 10 Virtual Machine
- Sysmon (System Monitor)
- Splunk Universal Forwarder
- Sysmon configuration file

### Network Requirements
- Network connectivity between VM and Splunk Enterprise
- Required ports open (default: 9997)

## Architecture Overview

```
Windows 10 VM
├── Sysmon (Event Logs)
├── Windows Event Logs
└── Splunk Universal Forwarder
    └── Forward to Splunk Enterprise (Windows Host)
        └── Splunk Indexing & Analysis
```

## Step 1: Splunk Enterprise Configuration

### 1.1 Configure Receiving Port

1. **Open Splunk Web Interface**
   - Navigate to `http://localhost:8000`
   - Login with admin credentials

2. **Configure Receiving**
   - Go to **Settings** → **Forwarding and receiving**
   - Click **Configure receiving**
   - Click **New**
   - Fill in:
     - Listen on this port: `9997`
     - Network: `0.0.0.0` (or specific IP)
   - Click **Save**

3. **Create Index for VM Logs**
   - Go to **Settings** → **Indexes**
   - Click **New**
   - Create index named `vm_windows_logs`
   - Configure settings:
     - Max size: `500000` MB
     - Retention: As per policy
   - Click **Save**

## Step 2: Windows 10 VM Setup

### 2.1 Enable Required Event Logs

1. **Open Event Viewer**
   - Press `Win + R`, type `eventvwr.msc`
   - Navigate to **Windows Logs**

2. **Verify Logs are Enabled**
   - Application
   - Security
   - System
   - Sysmon (will appear after Sysmon installation)

### 2.2 Configure Audit Policies (Optional)

1. **Open Group Policy Editor**
   - Press `Win + R`, type `gpedit.msc`
   - Navigate to: **Computer Configuration** → **Windows Settings** → **Security Settings** → **Advanced Audit Policy Configuration**

2. **Enable Detailed Auditing**
   - Account Logon: Success/Failure
   - Account Management: Success/Failure
   - Detailed Tracking: Success
   - DS Access: Success/Failure
   - Logon/Logoff: Success/Failure
   - Object Access: Success/Failure
   - Policy Change: Success/Failure
   - Privilege Use: Success/Failure
   - System: Success/Failure

## Step 3: Sysmon Installation

### 3.1 Download Sysmon

1. **Download Sysmon**
   - Visit: https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon
   - Download Sysmon ZIP file

2. **Create Sysmon Configuration**
   - Create file: `sysmon-config.xml`
   - Use recommended configuration from SwiftOnSecurity:
     ```bash
     # Download recommended configuration
     curl -o sysmon-config.xml https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml
     ```

### 3.2 Install Sysmon

1. **Open Command Prompt as Administrator**
   - Extract Sysmon ZIP to `C:\Tools\Sysmon\`
   - Navigate to extraction directory

2. **Install with Configuration**
   ```cmd
   # Install Sysmon with configuration
   sysmon.exe -accepteula -i sysmon-config.xml
   
   # Verify installation
   sysmon.exe -c --loglevel=Info
   ```

3. **Verify Sysmon Service**
   - Press `Win + R`, type `services.msc`
   - Look for "Sysmon" service
   - Ensure it's running

## Step 4: Universal Forwarder Installation

### 4.1 Download Universal Forwarder

1. **Download UF**
   - Visit: https://www.splunk.com/en_us/download/universal-forwarder.html
   - Download Windows Universal Forwarder

2. **Install Universal Forwarder**
   ```cmd
   # Run installer as Administrator
   splunkforwarder-9.x.x-xxxxx-x64-release.msi
   ```

3. **Installation Parameters**
   - Installation Directory: `C:\Program Files\SplunkUniversalForwarder\`
   - Credentials: Set admin password
   - Deployment Server: Leave blank
   - Receiving Indexer: Leave blank for now

### 4.2 Configure Universal Forwarder

1. **Open Command Prompt as Administrator**
   ```cmd
   # Navigate to Splunk bin directory
   cd "C:\Program Files\SplunkUniversalForwarder\bin\"
   ```

2. **Configure Forwarding**
   ```cmd
   # Set deployment server (if using)
   splunk set deploy-poll splunk-enterprise-ip:8089
   
   # Add forwarding to Splunk Enterprise
   splunk add forward-server splunk-enterprise-ip:9997
   
   # Enable Splunk to start at boot
   splunk enable boot-start
   
   # Restart Splunk forwarder
   splunk restart
   ```

## Step 5: Data Inputs Configuration

### 5.1 Configure Inputs.conf

1. **Navigate to UF Directory**
   - Location: `C:\Program Files\SplunkUniversalForwarder\etc\system\local\`

2. **Create/Edit inputs.conf**
   ```ini
   [default]
   host = win10-vm
   
   # Windows Event Logs
   [WinEventLog://Application]
   disabled = 0
   index = vm_windows_logs
   
   [WinEventLog://Security]
   disabled = 0
   index = vm_windows_logs
   
   [WinEventLog://System]
   disabled = 0
   index = vm_windows_logs
   
   [WinEventLog://Sysmon]
   disabled = 0
   index = vm_windows_logs
   
   [WinEventLog://Microsoft-Windows-Sysmon/Operational]
   disabled = 0
   index = vm_windows_logs
   
   # Performance Monitoring (Optional)
   [perfmon://Processor]
   disabled = 0
   index = vm_windows_logs
   object = Processor
   counters = % Processor Time; % User Time; % Privileged Time
   instances = *
   interval = 60
   useEnglishOnly=true
   
   [perfmon://Memory]
   disabled = 0
   index = vm_windows_logs
   object = Memory
   counters = Available MBytes; % Committed Bytes In Use
   instances = *
   interval = 60
   useEnglishOnly=true
   ```

### 5.2 Create props.conf and transforms.conf (Optional)

1. **Create props.conf** (`C:\Program Files\SplunkUniversalForwarder\etc\system\local\`)
   ```ini
   [source::WinEventLog]
   SHOULD_LINEMERGE = false
   LINE_BREAKER = ([\r\n]+)
   TRUNCATE = 0
   
   [WinEventLog://Microsoft-Windows-Sysmon/Operational]
   SHOULD_LINEMERGE = false
   LINE_BREAKER = ([\r\n]+)
   TRUNCATE = 0
   ```

## Step 6: Splunk Enterprise App Setup

### 6.1 Install Splunk App for Windows

1. **Download Splunk App for Windows**
   - Visit Splunkbase: https://splunkbase.splunk.com/app/742/
   - Download the app

2. **Install the App**
   - In Splunk Web, go to **Apps** → **Manage Apps**
   - Click **Install app from file**
   - Upload downloaded .spl file
   - Restart Splunk if required

### 6.2 Configure Splunk Enterprise

1. **Create Local inputs.conf** (`$SPLUNK_HOME/etc/system/local/`)
   ```ini
   [tcp://9997]
   disabled = 0
   ```

2. **Restart Splunk Enterprise**
   ```cmd
   # Windows Command Prompt
   cd "C:\Program Files\Splunk\bin\"
   splunk restart
   ```

## Step 7: Verification and Testing

### 7.1 Verify Data Flow

1. **Check Universal Forwarder Status**
   ```cmd
   # On VM
   cd "C:\Program Files\SplunkUniversalForwarder\bin\"
   splunk list forward-server
   splunk status
   ```

2. **Generate Test Events**
   - Create a test event in Event Log
   - Run a command to trigger Sysmon event
   - Check if events appear in Splunk

3. **Verify in Splunk Web**
   - Go to **Search & Reporting**
   - Run query: `index="vm_windows_logs" | head 10`
   - Run query: `index="vm_windows_logs" source="*Sysmon*" | head 10`

### 7.2 Create Basic Dashboards

1. **Sysmon Dashboard**
   - Go to **Dashboards** → **Create New Dashboard**
   - Add panels for:
     - Process Creation Events
     - Network Connections
     - File Creation Events
     - Security Events

2. **Sample Searches for Dashboard**
   ```sql
   # Process Creation
   index="vm_windows_logs" EventID=1 | table _time, host, process_name, command_line
   
   # Network Connections
   index="vm_windows_logs" EventID=3 | table _time, host, dest_ip, dest_port, process_name
   
   # Security Events
   index="vm_windows_logs" sourcetype="WinEventLog:Security" | timechart count by EventCode
   ```

## Step 8: Troubleshooting

### 8.1 Common Issues and Solutions

1. **UF Not Connecting**
   ```cmd
   # Check network connectivity
   telnet splunk-enterprise-ip 9997
   
   # Check UF logs
   cd "C:\Program Files\SplunkUniversalForwarder\bin\"
   splunk list forward-server
   splunk show splunkd-port
   ```

2. **No Data in Splunk**
   - Check if indexes are created
   - Verify inputs.conf configuration
   - Check UF internal logs
   - Verify event logs exist on VM

3. **Sysmon Events Not Appearing**
   ```cmd
   # Verify Sysmon is running
   Get-Service Sysmon
   
   # Check Sysmon operational log
   eventvwr.msc
   Navigate to: Applications and Services Logs → Microsoft → Windows → Sysmon → Operational
   ```

### 8.2 Useful Commands

```cmd
# Universal Forwarder Management
splunk status
splunk list forward-server
splunk restart
splunk help

# Check UF logs
cd "C:\Program Files\SplunkUniversalForwarder\var\log\splunk\"
```

```powershell
# PowerShell commands for verification
Get-WinEvent -ListLog *sysmon*
Get-Service | Where-Object {$_.Name -like "*splunk*"}
Test-NetConnection splunk-enterprise-ip -Port 9997
```

## Step 9: Monitoring and Maintenance

### 9.1 Regular Checks

1. **Monitor UF Health**
   - Check UF service status
   - Monitor queue sizes
   - Verify data ingestion

2. **Sysmon Maintenance**
   - Update Sysmon configuration periodically
   - Monitor Sysmon service
   - Review Sysmon logs for errors

### 9.2 Performance Considerations

1. **UF Resource Usage**
   - Monitor CPU and memory usage
   - Adjust batch sizes if needed
   - Consider indexing volume

2. **Network Impact**
   - Monitor network bandwidth
   - Consider compression
   - Schedule heavy transfers appropriately

## Conclusion

This comprehensive guide provides end-to-end instructions for monitoring Windows 10 VM logs using Splunk Enterprise with Sysmon integration. The setup includes:

- ✅ Splunk Enterprise receiving configuration
- ✅ Windows 10 VM preparation
- ✅ Sysmon installation and configuration
- ✅ Universal Forwarder deployment
- ✅ Data inputs configuration
- ✅ Verification and testing procedures
- ✅ Troubleshooting guidance

The system will now monitor security events, system logs, application logs, and detailed Sysmon events from your Windows 10 VM, providing comprehensive visibility into system activities.

For updates and additional configurations, refer to the official Splunk and Sysmon documentation.
