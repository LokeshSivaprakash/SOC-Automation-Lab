# Setup Guide

## Prerequisites
- Two Vultr Ubuntu 24.04 VPS (8GB RAM for Wazuh, 12GB for TheHive)
- VMware Workstation with Windows 11 VM
- Shuffle account at shuffler.io
- VirusTotal API key (free tier works)

---

## 1. Wazuh Manager Setup
```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
bash wazuh-install.sh -a
```

---

## 2. TheHive Setup
```bash
# Install dependencies
apt install -y openjdk-11-jre-headless cassandra elasticsearch

# Install TheHive
apt install -y thehive

# Start services in order
systemctl start cassandra
sleep 30
systemctl start elasticsearch
sleep 30
systemctl start thehive
```

---

## 3. Common Issues and Fixes

| Issue | Cause | Fix |
|---|---|---|
| Cassandra won't start | Stray character in yaml file | Check first line of cassandra.yaml |
| TheHive crashes on startup | Config syntax error | Check application.conf for stray characters |
| Elasticsearch refuses connections | SSL enabled by default | Set xpack.security.enabled: false |
| Wazuh agent never connects | Firewall blocking port 1514 | Open TCP/UDP 1514 on server and cloud firewall |
| TheHive not loading on browser | Port 9000 blocked | Open port 9000 on Vultr dashboard and UFW |

---

## 4. Wazuh Agent on Windows
```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.0-1.msi `
  -OutFile wazuh-agent.msi
msiexec /i wazuh-agent.msi /q WAZUH_MANAGER="YOUR_WAZUH_IP"
NET START WazuhSvc
```

---

## 5. Shuffle Workflow Setup
- Create a new workflow in Shuffle
- Add Webhook trigger and copy the webhook URL
- Paste webhook URL into Wazuh integration config
- Add SHA256 extraction step using Shuffle Tools
- Connect VirusTotal app with your API key
- Connect TheHive app with your TheHive URL and API key
- Add Email notification step
- Save and activate the workflow

---

## 6. Custom Wazuh Rule
Place this in `/var/ossec/etc/rules/local_rules.xml` on your Wazuh server:
```xml
<group name="mimikatz,credential_access,">
  <rule id="100002" level="15">
    <if_group>syscheck</if_group>
    <field name="win.eventdata.originalFileName" 
           type="pcre2">(?i)mimikatz</field>
    <description>Mimikatz Usage Detected</description>
    <mitre>
      <id>T1003</id>
    </mitre>
  </rule>
</group>
```

---

## 7. Testing the Pipeline
1. Start all services on both Vultr servers
2. Make sure Wazuh agent is active on Windows VM
3. Run Mimikatz on the Windows VM
4. Watch Wazuh dashboard for the alert
5. Check Shuffle for the workflow execution
6. Confirm TheHive case was auto created
7. Check your email for the analyst notification

---

## 8. Useful Commands
```bash
# Check all service statuses
systemctl status cassandra
systemctl status elasticsearch
systemctl status thehive
systemctl status wazuh-manager

# Check Wazuh connected agents
/var/ossec/bin/agent_control -l

# Check TheHive logs
tail -f /var/log/thehive/application.log

# Check Cassandra logs
tail -f /var/log/cassandra/system.log

# Check Elasticsearch logs
tail -f /var/log/elasticsearch/elasticsearch.log
```
