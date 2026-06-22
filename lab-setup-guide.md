# SentinelX — Complete Lab Setup Guide

> This document covers the full journey of building the SentinelX SOC lab from scratch — including failed attempts, problems encountered, and how each was solved. It is written as an honest account of the build process, not a polished tutorial that assumes everything works first time.

---

## Table of Contents

1. [Project Goal](#1-project-goal)
2. [Attempt 1 — Wazuh Cloud Trial (Failed)](#2-attempt-1--wazuh-cloud-trial-failed)
3. [Attempt 2 — Ubuntu 26.04 Self-Hosted (Failed)](#3-attempt-2--ubuntu-2604-self-hosted-failed)
4. [Attempt 3 — Official Wazuh OVA (Succeeded)](#4-attempt-3--official-wazuh-ova-succeeded)
5. [Fixing Service Startup Issues](#5-fixing-service-startup-issues)
6. [Accessing the Dashboard](#6-accessing-the-dashboard)
7. [Enrolling the Windows Agent](#7-enrolling-the-windows-agent)
8. [Configuring Windows Audit Policy](#8-configuring-windows-audit-policy)
9. [Verifying the Full Pipeline](#9-verifying-the-full-pipeline)
10. [Scenario 1 — Brute Force Attack](#10-scenario-1--brute-force-attack)
11. [Scenario 2 — Suspicious PowerShell Execution](#11-scenario-2--suspicious-powershell-execution)
12. [Scenario 3 — Unauthorized Account Creation](#12-scenario-3--unauthorized-account-creation)
13. [Persistent Issues & Known Gotchas](#13-persistent-issues--known-gotchas)

---

## 1. Project Goal

The goal was to build a portfolio-ready SOC analyst lab that demonstrates real, hands-on skills — not just tool familiarity. The requirements were:

- A real SIEM running on real infrastructure (not a cloud sandbox someone else manages)
- A real Windows endpoint enrolled and actively monitored
- Real attack simulations — not just clicking through pre-built demos
- Formal incident reports for each scenario, written the way a real SOC analyst would
- Evidence of detection engineering — not just using default rules, but understanding and extending them

The platform was named **SentinelX** to present it as a cohesive project rather than a collection of loosely related exercises.

---

## 2. Attempt 1 — Wazuh Cloud Trial (Failed)

### What was tried
Signed up for Wazuh's free cloud trial. Created an environment named "HawkEye." The environment reached "Ready" status successfully, and the dashboard URL was accessible.

### What went wrong
Login credentials consistently failed despite multiple resets. After investigation and contacting Wazuh support (Julieta Rodón), the root cause was confirmed:

> **Wazuh Cloud trials are restricted to business and corporate email domains. College and personal email addresses are flagged and the trial is disabled.**

The trial could not be reinstated for educational use. A paid training discount was offered (declined). Wazuh support recommended the self-hosted OVA as an alternative.

### Lesson
Wazuh Cloud is not viable for student/personal use. Self-hosted is the right path for a lab environment anyway — it gives full control over configuration and better demonstrates infrastructure skills.

---

## 3. Attempt 2 — Ubuntu 26.04 Self-Hosted (Failed)

### What was tried
Built a fresh VM in VMware Workstation Pro with the following specs:
- **OS:** Ubuntu Server 26.04 (latest available at the time)
- **RAM:** 4GB
- **CPU:** 2 cores
- **Disk:** 40GB
- **Networking:** NAT

Ran Wazuh's official all-in-one install script:
```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash wazuh-install.sh -a -i
```

The Wazuh indexer and manager installed successfully. The dashboard installation repeatedly failed.

### What went wrong
The dashboard install consistently errored during package extraction with a dpkg error related to a `prismjs` file. After research, the root cause was identified:

> **Ubuntu 26.04 is too new. Wazuh 4.x officially supports up to Ubuntu 22.04. Ubuntu 26.04 has library and dependency changes that break the dashboard package extraction.**

Multiple workarounds were attempted (forcing dpkg, manual extraction, dependency pinning) — none resolved the issue cleanly. The decision was made to abandon this approach entirely rather than continue fighting an unsupported OS combination.

### Lesson
Always check official support matrices before picking an OS for a self-hosted security tool. Newer is not always better — Wazuh explicitly lists supported distributions and Ubuntu 26.04 was not on the list.

---

## 4. Attempt 3 — Official Wazuh OVA (Succeeded)

### Why OVA
The official Wazuh OVA (Open Virtual Appliance) ships as a pre-built VM image with all components installed and configured. It runs on Amazon Linux 2023, which Wazuh fully supports and tests against. This eliminates the OS compatibility problem entirely.

### Download
Downloaded `wazuh-4.x.ova` from the official Wazuh documentation:
```
https://documentation.wazuh.com/current/deployment-options/virtual-machine/virtual-machine.html
```

### Import into VMware
1. VMware Workstation Pro → File → Open → select the `.ova` file
2. Accept the import prompts
3. Before powering on, adjust VM resources to fit the host machine (8GB RAM laptop):
   - RAM: **4GB** (down from default 8GB)
   - CPU: **2 cores** (down from default 4)
   - Disk: left at default (~40GB)
   - Network adapter: set to **NAT**

### First boot
Powered on — clean boot, no errors. No AMD CPU compatibility warnings.

Logged in at the VM console with default OVA credentials:
- **Username:** `wazuh-user`
- **Password:** `wazuh`

### Finding the VM IP
```bash
ip a
```
Output showed the NAT interface with IP: **192.168.136.131**

This IP is used for all subsequent connections — dashboard access, SSH, agent enrollment.

---

## 5. Fixing Service Startup Issues

### Problem
After the first boot, the Wazuh dashboard was not accessible. Investigating service status revealed that all three core services (indexer, manager, dashboard) were either inactive or failed:

```bash
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-dashboard
```

The indexer and manager both showed timeout errors — they had failed to start because systemd's default startup timeout (90 seconds) was too short for OpenSearch (the indexer) to fully initialize.

### Root cause
Wazuh's indexer is built on OpenSearch, a Java-based search engine that takes significantly longer than 90 seconds to start on a resource-constrained VM. Systemd killed the process as "failed" before it could finish initializing.

### Fix — Apply TimeoutStartSec override to both services

**This must be done via direct shell writes, not nano** (see [Known Gotchas](#13-persistent-issues--known-gotchas) for why).

For wazuh-indexer:
```bash
sudo mkdir -p /etc/systemd/system/wazuh-indexer.service.d
echo -e "[Service]\nTimeoutStartSec=300" | sudo tee /etc/systemd/system/wazuh-indexer.service.d/override.conf
```

For wazuh-manager:
```bash
sudo mkdir -p /etc/systemd/system/wazuh-manager.service.d
echo -e "[Service]\nTimeoutStartSec=300" | sudo tee /etc/systemd/system/wazuh-manager.service.d/override.conf
```

Reload systemd and start services in the correct order:
```bash
sudo systemctl daemon-reload
sudo systemctl start wazuh-indexer
# Wait ~60-90 seconds for indexer to fully initialize
sudo systemctl start wazuh-manager
sudo systemctl start wazuh-dashboard
```

**Critical:** Always start indexer first. The manager depends on the indexer being ready. Starting manager before indexer will cause it to fail.

### Verifying services are up
```bash
sudo systemctl status wazuh-indexer --no-pager
sudo systemctl status wazuh-manager --no-pager
sudo systemctl status wazuh-dashboard --no-pager
```

All three should show `Active: active (running)`.

---

## 6. Accessing the Dashboard

### URL
```
https://192.168.136.131
```

Accept the browser's self-signed certificate warning (expected for a local lab).

### Credentials
- **Username:** `admin`
- **Password:** `admin`

### First login issue
On the first login attempt after a fresh OVA boot, the dashboard may show a health check page with "No API available to connect." This happens when the manager API hasn't fully initialized yet.

**Fix:** Wait 60 seconds after the manager shows `active (running)`, then:
1. Go to Settings → API Connections
2. Click the refresh icon next to the default API entry
3. Status should change to "Online"

If it shows ERROR3099 (wazuh-modulesd failed), restart the manager:
```bash
sudo systemctl restart wazuh-manager
```

Then retry the API connection refresh.

---

## 7. Enrolling the Windows Agent

### Goal
Enroll the Windows 11 host (the same machine running VMware) as a monitored Wazuh agent, named "HawkEye."

### Steps

**In the Wazuh dashboard:**
1. Navigate to Agents → Deploy new agent
2. Select **Windows** as the OS
3. Enter server address: `192.168.136.131`
4. Set agent name: `HawkEye`
5. Copy the generated PowerShell install command

**On the Windows host (PowerShell as Administrator):**

Paste and run the generated install command — it downloads and installs the Wazuh MSI package silently.

Then start the agent service:
```powershell
NET START Wazuh
```

### Verification
Back in the dashboard → Agents page:
- Agent name: **HawkEye**
- Agent ID: **001**
- Status: **Active** (green dot)
- OS: Windows 11

The agent registers with the manager automatically using the enrollment key embedded in the install command.

### Note on agent persistence
The Wazuh agent service on Windows is set to start automatically. If the Windows machine is rebooted, the agent reconnects to the manager automatically once the manager is back up — no manual restart needed on the Windows side.

---

## 8. Configuring Windows Audit Policy

Full command-line logging must be explicitly enabled on Windows — it is not on by default. This is essential for detecting PowerShell obfuscation (Scenario 2) and tracking process execution chains.

### Enable Process Creation logging with full command line

Run in PowerShell as Administrator:

```powershell
# Enable Process Creation audit events (Event ID 4688)
auditpol /set /subcategory:"Process Creation" /success:enable

# Enable full command-line capture in Event ID 4688
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" `
  /v ProcessCreationIncludeCmdLine_Enabled /t REG_DWORD /d 1 /f
```

### Enable Logon/Logoff logging

```powershell
# Event IDs 4624 (success), 4625 (failure), 4634 (logoff)
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Logoff" /success:enable
```

### Enable Account Management logging

```powershell
# Event IDs 4720 (account created), 4722 (account enabled),
# 4738 (account changed), 4732 (added to group)
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
```

### Verify audit policy is active

```powershell
auditpol /get /category:*
```

Look for "Process Creation", "Logon", and "User Account Management" showing Success and/or Failure enabled.

---

## 9. Verifying the Full Pipeline

Before running any attack simulations, the end-to-end pipeline was verified with a single deliberate test:

### Test: Single failed login

```powershell
runas /user:definitelynotarealuser cmd
# Enter any wrong password when prompted
```

### Expected result
In Wazuh dashboard → Threat Hunting → Events:
- **Event ID:** 4625 (Logon Failure)
- **Rule:** 60122
- **Description:** Logon Failure — Unknown user or bad password
- **Agent:** HawkEye

This confirmed the full chain is working:
```
Windows Security Event → Wazuh Agent → Wazuh Manager → OpenSearch Indexer → Dashboard
```

Once this test passed, the lab was considered ready for scenarios.

---

## 10. Scenario 1 — Brute Force Attack

### Objective
Simulate a brute force credential attack and confirm Wazuh's correlation engine detects the pattern as distinct from isolated failed logins.

### Simulation

Run in PowerShell as Administrator on the Windows host:

```powershell
for ($i = 1; $i -le 10; $i++) {
    $pass = "wrongpassword$i"
    net use \\localhost\IPC$ /user:fakeadminuser $pass 2>$null
    Start-Sleep -Milliseconds 500
}
```

This sends 10 rapid failed authentication attempts against a non-existent account (`fakeadminuser`) with different passwords each time — mimicking an automated password spraying tool.

### Detection result

Wazuh fired a **correlation alert** (not just individual event logging):

| Field | Value |
|-------|-------|
| Rule ID | 60204 |
| Description | Multiple Windows Logon Failures |
| Level | 10 |
| MITRE | T1110 — Brute Force |
| Tactic | Credential Access |

### Key investigation findings

- **Source IP:** `::1` (IPv6 loopback) — confirms the attack came from the local machine, not an external source
- **Auth protocol:** NTLM
- **Logon Type:** 3 (Network logon)
- **Sequential source ports:** Incrementing port numbers confirmed automated/scripted timing rather than manual attempts

### Incident report
→ See `incidents/INC-001-Brute-Force-Report.docx`

---

## 11. Scenario 2 — Suspicious PowerShell Execution

### Objective
Simulate an obfuscated PowerShell command using Base64 encoding — a common attacker defence evasion technique — and investigate both the detection gap in the default ruleset and the custom rule written to close it.

### Simulation

Run in PowerShell as Administrator on the Windows host:

```powershell
$cmd = 'Get-Process | Out-File C:\Windows\Temp\sysinfo_cache.txt; Write-Host "Recon complete"'
$bytes = [System.Text.Encoding]::Unicode.GetBytes($cmd)
$encoded = [Convert]::ToBase64String($bytes)
powershell.exe -NoProfile -WindowStyle Hidden -EncodedCommand $encoded
```

This mimics the attacker pattern of:
- `-NoProfile` — reduces logging hooks
- `-WindowStyle Hidden` — no visible console window
- `-EncodedCommand` — payload hidden behind Base64 encoding

### Initial detection result (default ruleset)

The event was captured, but only as a generic Level 3 alert:

| Field | Value |
|-------|-------|
| Rule ID | 67027 |
| Description | A process was created. |
| Level | 3 (Informational) |

Searching for `*EncodedCommand*` in the commandLine field returned only 2 hits out of 178 total process creation events — both at Level 3, both indistinguishable from legitimate activity in the alert queue.

**This is a detection gap.** The SIEM captured the event but did not flag it as suspicious.

### Detection gap remediation — Custom rule

SSH into the Wazuh server:
```bash
ssh wazuh-user@192.168.136.131
```

First, identify the parent rule structure:
```bash
sudo grep -rn "67027" /var/ossec/ruleset/rules/
# Returns: /var/ossec/ruleset/rules/0955-WEF-baseline_rules.xml:188
sudo sed -n '180,200p' /var/ossec/ruleset/rules/0955-WEF-baseline_rules.xml
```

This confirms rule 67027 uses `<if_sid>60103</if_sid>` and matches Event ID 4688. Our custom rule chains onto it.

Append the custom rule to local_rules.xml:
```bash
sudo tee -a /var/ossec/etc/rules/local_rules.xml > /dev/null << 'EOF'

<group name="powershell,windows,sentinelx,">
  <rule id="100002" level="12">
    <if_sid>67027</if_sid>
    <field name="win.eventdata.commandLine" type="pcre2">(?i)-(enc|encodedcommand)\b</field>
    <description>Suspicious PowerShell execution: Base64-encoded/obfuscated command detected</description>
    <mitre>
      <id>T1059.001</id>
      <id>T1027</id>
    </mitre>
    <group>powershell_obfuscation,</group>
  </rule>
</group>
EOF
```

Restart the manager to load the new rule:
```bash
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager --no-pager
```

### Re-simulation and validation

Re-run the encoded PowerShell command on the Windows host. In Threat Hunting, search:
```
rule.id: 100002
```

Expected result:
- **1 alert at Level 12**
- MITRE ATT&CK: T1059.001 (PowerShell) + T1027 (Obfuscated Files)
- Agent: HawkEye

### Payload decoding (analyst step)

Decode the captured Base64 payload to confirm what the command would have done:
```powershell
[System.Text.Encoding]::Unicode.GetString(
  [Convert]::FromBase64String("RwBlAHQALQBQAHIAbwBjAGUAcwBzACAAfAAgAE8AdQB0...")
)
# Output: Get-Process | Out-File C:\Windows\Temp\sysinfo_cache.txt; Write-Host "Recon complete"
```

This confirms: process enumeration written to a temp file — basic reconnaissance, consistent with early-stage attacker behaviour.

### Incident report
→ See `incidents/INC-002-Suspicious-PowerShell-Report.docx`

---

## 12. Scenario 3 — Unauthorized Account Creation

### Objective
Simulate an attacker creating a backdoor local user account with a deceptive service-account name and immediately escalating it to Administrator — a classic persistence technique.

### Simulation

Run in PowerShell as Administrator on the Windows host:

```powershell
# Step 1: Create the backdoor account
New-LocalUser -Name "svc_backup" `
  -Password (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force) `
  -FullName "Backup Service" `
  -Description "System backup account"

# Step 2: Escalate to Administrator
Add-LocalGroupMember -Group "Administrators" -Member "svc_backup"
```

The name `svc_backup` mimics a legitimate backup service account — a deliberate social engineering choice to evade casual inspection.

### Detection result (native ruleset — no custom rule needed)

Wazuh fired 4 alerts within a single second:

| Rule ID | Description | Event ID | Level |
|---------|-------------|----------|-------|
| 60109 | User account enabled or created | 4722 | 8 |
| 60109 | User account enabled or created | 4722 | 8 |
| 60110 | User account changed | 4738 | 8 |
| 60110 | User account changed | 4738 | 8 |

MITRE ATT&CK was **pre-mapped by the native ruleset** — no custom rule required:
- **T1098** — Account Manipulation
- **Tactic:** Persistence

Compliance frameworks auto-tagged: GDPR (IV_35.7.d), HIPAA (164.312.a.2.I), PCI-DSS (8.1.2, 10.2.5), NIST 800-53 (AC.2, IA.4, AU.14, AC.7)

### Investigation

Search in Threat Hunting:
```
data.win.eventdata.targetUserName: svc_backup
```

Expand the event documents to confirm:
- Creator: `lenovo` (ESHAAN domain)
- Target account: `svc_backup`
- Target SID: `S-1-5-21-2327742703-2673396736-1529539244-1025`
- Both creation and group modification events captured

### Containment

After investigation, remove the simulated attacker account:
```powershell
Remove-LocalGroupMember -Group "Administrators" -Member "svc_backup"
Remove-LocalUser -Name "svc_backup"
```

Clean up recon files from Scenario 2:
```powershell
Remove-Item "C:\Windows\Temp\sysinfo_cache.txt" -ErrorAction SilentlyContinue
Remove-Item "C:\Windows\Temp\sysinfo_cache2.txt" -ErrorAction SilentlyContinue
```

### Incident report
→ See `incidents/INC-003-Account-Creation-Report.docx`

---

## 13. Persistent Issues & Known Gotchas

### TimeoutStartSec overrides don't always persist
After an unclean reboot or hard shutdown, the systemd override files may be lost. Always check:
```bash
cat /etc/systemd/system/wazuh-indexer.service.d/override.conf
cat /etc/systemd/system/wazuh-manager.service.d/override.conf
```

If either returns "No such file or directory", reapply the overrides using the commands in Section 5.

### Never use nano in the VMware console
VMware Workstation intercepts Ctrl+O (the nano save shortcut), causing "temporary file is empty" errors and file write failures. This applies when working directly in the VMware console window.

**Always use:**
```bash
echo -e "content" | sudo tee /path/to/file
# or
sudo tee /path/to/file > /dev/null << 'EOF'
content
EOF
```

**Or better:** SSH in from Windows PowerShell where clipboard and keyboard shortcuts work normally:
```powershell
ssh wazuh-user@192.168.136.131
```

### wazuh-modulesd segfault after VM resume
After suspending and resuming the VM, the vulnerability scanner submodule may crash. Symptom: dashboard API shows ERROR3099. Fix: `sudo systemctl restart wazuh-manager`.

### SSH disconnects on first connection attempt
First SSH connection after a manager restart sometimes disconnects immediately after showing the Wazuh ASCII banner. This is a one-off timing issue — retry the connection and it will stay stable.

### VM IP may shift on reboot
If DHCP reassigns the VM a different IP on boot, update your connection string. Check with `ip a` on the VM console. To make the IP static, edit the VMware DHCP reservation or configure a static IP in the VM's network settings.

---

## Summary

| Phase | Status | Key Outcome |
|-------|--------|-------------|
| Wazuh Cloud | ❌ Failed | College email blocked — pivoted to self-hosted |
| Ubuntu 26.04 | ❌ Failed | OS too new — Wazuh incompatibility |
| Wazuh OVA | ✅ Succeeded | Stable platform on Amazon Linux 2023 |
| Agent enrollment | ✅ Succeeded | HawkEye (ID 001) active |
| Audit policy | ✅ Succeeded | Full command-line logging enabled |
| Scenario 1 | ✅ Complete | Rule 60204, Level 10, T1110 |
| Scenario 2 | ✅ Complete | Custom rule 100002, Level 12, T1059.001/T1027 |
| Scenario 3 | ✅ Complete | Rules 60109/60110, Level 8, T1098 |

---

*Built by Eshaan — SentinelX SOC Investigation Platform*
