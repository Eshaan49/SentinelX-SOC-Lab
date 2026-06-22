# SentinelX Lab — Infrastructure Notes

## Setup Summary

| Component | Details |
|-----------|---------|
| Wazuh Server | OVA v4.x, Amazon Linux 2023, VMware Workstation Pro |
| VM Resources | 4GB RAM, 2 CPU cores, 40GB disk |
| Networking | NAT — VM IP: 192.168.136.131 |
| Dashboard | https://192.168.136.131 (admin/admin) |
| SSH | wazuh-user@192.168.136.131 (password: wazuh) |
| Windows Agent | HawkEye, Agent ID 001, Windows 11 |

---

## Known Issues & Workarounds

### 1. Systemd Timeout on Service Start
**Problem:** wazuh-indexer and wazuh-manager fail with timeout errors on boot/restart. OpenSearch (indexer) takes longer than systemd's default timeout to initialize.

**Fix:** Apply TimeoutStartSec=300 override to both services:
```bash
sudo mkdir -p /etc/systemd/system/wazuh-indexer.service.d
echo -e "[Service]\nTimeoutStartSec=300" | sudo tee /etc/systemd/system/wazuh-indexer.service.d/override.conf

sudo mkdir -p /etc/systemd/system/wazuh-manager.service.d
echo -e "[Service]\nTimeoutStartSec=300" | sudo tee /etc/systemd/system/wazuh-manager.service.d/override.conf

sudo systemctl daemon-reload
```

**Note:** These overrides may not persist across unclean reboots — reapply if services fail after restart.

**Start order:** Always start indexer first, then manager, then dashboard:
```bash
sudo systemctl start wazuh-indexer
sudo systemctl start wazuh-manager
sudo systemctl start wazuh-dashboard
```

---

### 2. VMware Console Keyboard Conflict (nano / Ctrl+O)
**Problem:** VMware Workstation intercepts Ctrl+O, making it impossible to save files in nano when working directly in the VM console window.

**Workaround:** Never use nano or systemctl edit for file writes. Use direct shell writes instead:
```bash
# Instead of nano, use tee:
echo -e "content" | sudo tee /path/to/file

# For multi-line files, use heredoc via SSH:
sudo tee -a /path/to/file > /dev/null << 'EOF'
multi
line
content
EOF
```

**Better solution:** SSH into the VM from Windows PowerShell — clipboard paste works normally:
```powershell
ssh wazuh-user@192.168.136.131
```

---

### 3. wazuh-modulesd Segfault on Resume
**Problem:** After VM suspend/resume, wazuh-modulesd may segfault in the vulnerability-scanner submodule. This blocks the dashboard API connection.

**Symptom:** Dashboard health check shows ERROR3099 — wazuh-modulesd->failed.

**Fix:** Restart the manager:
```bash
sudo systemctl restart wazuh-manager
```

This is a known rough edge with the vulnerability scanner on this OVA — does not affect core detection, alerting, or agent connectivity.

---

### 4. Dashboard API Connection Lost After Restart
**Problem:** After manager restart, dashboard may show "No API available to connect."

**Fix:** Go to Settings → API Connections → click the refresh icon next to the default entry. Usually resolves within 30 seconds of manager coming fully online.

---

## Windows Audit Policy Configuration

Applied via auditpol on the HawkEye endpoint:

```powershell
# Enable Process Creation logging (Event ID 4688)
auditpol /set /subcategory:"Process Creation" /success:enable

# Enable full command-line capture
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" /v ProcessCreationIncludeCmdLine_Enabled /t REG_DWORD /d 1 /f

# Enable Logon/Logoff (Event IDs 4624, 4625, 4634)
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Logoff" /success:enable

# Enable Account Management (Event IDs 4720, 4722, 4738, 4732)
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
```

---

## Wazuh Agent Enrollment (Windows)

1. Dashboard → Agents → Deploy new agent
2. Select Windows, enter server IP (192.168.136.131), name agent "HawkEye"
3. Copy generated MSI install command, run in PowerShell as Administrator
4. Start service: `NET START Wazuh`
5. Confirm in dashboard: Agents → HawkEye → Active

---

## File Locations (Wazuh Server)

| Purpose | Path |
|---------|------|
| Custom rules | /var/ossec/etc/rules/local_rules.xml |
| Default rules | /var/ossec/ruleset/rules/ |
| Wazuh config | /var/ossec/etc/ossec.conf |
| Alert logs | /var/ossec/logs/alerts/alerts.log |
| Systemd overrides | /etc/systemd/system/wazuh-*.service.d/override.conf |
