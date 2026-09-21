# Wazuh-Security-Monitoring-Lab

# SOC Lab – Wazuh Security Monitoring

> This project documents what I actually configured, tested, monitored, and investigated in my SOC lab. The screenshots included in this repository are evidence of the configurations, commands, detections, dashboards, and investigations documented below.

---

# Overview

In this project, I built a SOC monitoring lab using Wazuh with Windows and Linux systems.

During the project, I:

* Configured Wazuh agents.
* Monitored Windows Security events.
* Configured File Integrity Monitoring.
* Created and modified Wazuh detection rules.
* Enabled and investigated the Windows Guest account.
* Created and investigated Windows user accounts.
* Investigated account deletion events.
* Investigated successful Windows logons.
* Investigated Windows group membership changes.
* Checked Sysmon on Windows.
* Checked Sysmon on Linux.
* Configured a Linux Wazuh agent.
* Tested file modification monitoring.
* Created a SOC dashboard.

---

# Lab Environment

## Windows

My Windows environment was joined to:

```text
corp.local
```

I configured a Wazuh Windows agent with the name:

```text
KAVIN-DC01
```

The agent was configured to communicate with my Wazuh server over:

```text
Protocol: TCP
Port: 1514
Server: [REDACTED]
```

The enrollment configuration included:

```xml
<enrollment>
    <agent_name>KAVIN-DC01</agent_name>
</enrollment>
```

---

# 1. Wazuh Agent Configuration

I inspected and modified the Windows Wazuh configuration file:

```text
ossec.conf
```

I configured the agent to communicate with the Wazuh server using TCP on port `1514`.

## Re-registering the Windows Agent

I created a PowerShell script that:

1. Loaded the Wazuh configuration.
2. Checked for an existing enrollment block.
3. Removed the old enrollment block when present.
4. Created a new enrollment block.
5. Set the agent name to `KAVIN-DC01`.
6. Saved the configuration.
7. Stopped the Wazuh service.
8. Removed `client.keys`.
9. Started the Wazuh service again.

The paths I used were:

```powershell
$wazuhConf = "C:\Program Files (x86)\ossec-agent\ossec.conf"

$clientKeys = "C:\Program Files (x86)\ossec-agent\client.keys"
```

I loaded the configuration using:

```powershell
$xml = Get-Content $wazuhConf

$clientNode = $xml.SelectSingleNode("//client")
```

I checked for an old enrollment block and removed it when present:

```powershell
$oldEnrollment = $xml.SelectSingleNode("//enrollment")

if ($oldEnrollment) {
    $oldEnrollment.ParentNode.RemoveChild($oldEnrollment) | Out-Null
}
```

I configured the agent name:

```powershell
$agentNameNode.InnerText = "KAVIN-DC01"
```

To force the agent to register again, I stopped the service:

```powershell
Stop-Service WazuhSvc
```

I removed the existing client keys:

```powershell
Remove-Item $clientKeys -Force
```

I then started the service again:

```powershell
Start-Service WazuhSvc
```

---

# 2. File Integrity Monitoring

I configured Wazuh File Integrity Monitoring on Windows.

The `ossec.conf` configuration included:

```xml
<syscheck>
    <disabled>no</disabled>
    <frequency>43200</frequency>
</syscheck>
```

The configuration included monitoring Windows locations and executables such as:

```text
regedit.exe
system.ini
win.ini
at.exe
attrib.exe
cacls.exe
cmd.exe
eventcreate.exe
ftp.exe
net.exe
WMIC.exe
powershell.exe
winrm.vbs
```

## File Monitoring Test

I created a test directory and a test file to use during my file monitoring tests.

The file contained test data indicating that it should not be tampered with and was intended for file integrity monitoring.

---

# 3. Linux Wazuh Agent and File Monitoring

I also configured a Linux Wazuh agent.

I edited:

```bash
/var/ossec/etc/ossec.conf
```

The File Integrity Monitoring configuration included:

```xml
<syscheck>
    <disabled>no</disabled>
    <frequency>43200</frequency>
    <scan_on_start>yes</scan_on_start>

    <directories>/etc,/usr/bin,/usr/sbin</directories>
    <directories>/bin,/sbin,/boot</directories>
</syscheck>
```

I restarted the Wazuh agent:

```bash
sudo systemctl restart wazuh-agent
```

I checked the agent status:

```bash
sudo systemctl status wazuh-agent
```

The service showed:

```text
Active: active (running)
```

During my configuration work, I also encountered warnings including:

```text
Invalid attribute 'type' for 'ignore' option
```

and:

```text
Configuration error at 'etc/ossec.conf'
```

I used these messages while troubleshooting my Linux Wazuh agent configuration.

## Linux File Modification Test

I created and modified a test file under:

```text
/opt/company-data/
```

The file was initially created with test content and was then modified as part of my file monitoring test.

---

# 4. Linux Network and Agent Connectivity

I checked the Linux network configuration using:

```bash
ip a
```

I restarted the Linux Wazuh agent using:

```bash
sudo systemctl restart wazuh-agent
```

I also configured firewall access for Wazuh agent communication:

```bash
sudo ufw allow 1514/tcp
sudo ufw allow 1514/udp
```

---

# 5. Windows Sysmon

I checked whether Sysmon was running on my Windows system.

I used:

```powershell
Get-Service sysmon*
```

The output showed:

```text
Running
Sysmon64
```

I also checked the Sysmon process:

```powershell
Get-Process sysmon*
```

The output showed:

```text
Sysmon64
```

This confirmed that Sysmon was running on the Windows system.

---

# 6. Linux Sysmon

I also checked Sysmon on Linux.

I used:

```bash
sudo systemctl status sysmon
```

The output showed:

```text
sysmon.service - Sysmon event logger
```

and:

```text
Active: active (running)
```

The output also showed Sysmon events including:

```text
Event ID 1
Event ID 5
```

---

# 7. Wazuh Dashboard

I created a dashboard named:

```text
MYDFIR-Kavin Basic SOC Activity
```

The dashboard included a visualization for:

## Failed Windows Logons

The dashboard showed failed Windows logon activity during the selected time period.

## Windows Account Changes Over Time

I created a time-series visualization showing Windows events over time.

The visible event IDs included:

```text
4673
4688
4689
1
13
4634
4624
22
4957
539
4625
7036
```

---

# 8. Custom Wazuh Rules

I worked directly with:

```text
local_rules.xml
```

I created and tested custom detection rules.

---

# 9. Guest Account Enabled Detection

I created a custom Wazuh rule to detect the Windows Guest account being enabled.

The rule I created was:

```xml
<group name="windows, windows_security, account_changed, adduser">

  <rule id="100200" level="12">

    <!-- Changed parent ID to 60000 (Generic Windows Security Event)
         to avoid dropping 4722 logs -->

    <if_sid>60000</if_sid>

    <field name="win.system.eventId">*4722$</field>

    <field name="win.eventdata.targetUserName">*Guest$</field>

    <description>
      MYDFIR-Kavin Windows Guest account was enabled.
    </description>

    <mitre>
      <id>T1078</id>
    </mitre>

    <group>
      windows,
      windows_account_management,
      account_enabled,
      guest_account,
    </group>

  </rule>

</group>
```

During testing, I changed the parent SID to:

```text
60000
```

I documented the reason in the rule:

```text
Changed parent ID to 60000 (Generic Windows Security Event)
to avoid dropping 4722 logs
```

---

# 10. Testing the Guest Account Detection

I enabled the Windows Guest account using:

```powershell
net user Guest /active:yes
```

The command completed successfully.

I checked the Guest account:

```powershell
net user Guest
```

The output showed:

```text
Account active Yes
```

I later disabled the account:

```powershell
net user Guest /active:no
```

## Event Generated

I investigated:

```text
Event ID 4722
```

My Wazuh Discover query was:

```text
data.win.eventdata.targetUserName: Guest
AND
data.win.system.eventID: 4722
```

The event showed:

```text
A user account was enabled.
```

The event fields showed:

```text
Target Account Name: Guest
```

and:

```text
Subject Account Name: Administrator
```

I then filtered Wazuh Discover using my custom rule ID.

My rule fired with the description:

```text
MYDFIR-Kavin Windows Guest account was enabled.
```

---

# 11. SSH Authentication Failure Rule

I also worked with a Wazuh rule for repeated SSH authentication failures.

The rule I used was:

```xml
<group name="local, syslog, sshd, authentication_failed">

  <rule id="100101"
        level="10"
        frequency="3"
        timeframe="120">

    <if_matched_sid>5760</if_matched_sid>

    <same_source_ip />

    <description>
      Multiple SSH login failures observed from the same source IP
    </description>

    <mitre>
      <id>T1118</id>
    </mitre>

    <group>
      authentication_failed,
      ssh_bruteforce,
      credential_access,
    </group>

  </rule>

</group>
```

The detection logic was:

```text
3 matching events
within 120 seconds
from the same source IP
```

---

# 12. Windows Security Event Investigation

I used Wazuh Discover to search for and inspect Windows Security events.

The Event IDs I investigated included:

| Event ID | Activity Investigated                          |
| -------- | ---------------------------------------------- |
| 4722     | Guest account enabled                          |
| 4720     | User account creation                          |
| 4726     | User account deletion                          |
| 4732     | Member added to a security-enabled local group |
| 4624     | Successful logon                               |
| 4634     | Logoff activity                                |

---

# 13. Event ID 4720 – User Account Created

I searched Wazuh Discover for:

```text
4720
```

and filtered the results to my Windows endpoint.

The events showed account creation activity.

I inspected fields including:

```text
data.win.eventdata.samAccountName
data.win.eventdata.targetUserName
data.win.eventdata.targetDomainName
data.win.eventdata.subjectUserSid
```

---

# 14. Event ID 4726 – User Account Deleted

I searched Wazuh Discover for:

```text
4726
```

and filtered the results to my Windows endpoint.

The event message showed:

```text
A user account was deleted.
```

I inspected the event fields to identify the account and the account that performed the action.

---

# 15. Event ID 4624 – Successful Logon

I searched Wazuh Discover for:

```text
4624
```

and filtered the results to my Windows endpoint.

The Wazuh event message showed:

```text
An account was successfully logged on.
```

I inspected fields including:

```text
Security ID
Account Name
Account Domain
Logon Type
New Logon
Network Information
```

The event shown in my investigation included:

```text
Logon Type: 3
```

---

# 16. Event ID 4732 – Group Membership Change

I searched Wazuh Discover for:

```text
4732
```

and filtered the results to my Windows endpoint.

I inspected fields including:

```text
data.win.eventdata.targetUserName
data.win.eventdata.memberSid
data.win.eventdata.subjectUserName
data.win.eventdata.targetDomainName
data.win.system.eventID
```

The event showed:

```text
Target User Name: Administrators
```

and:

```text
Subject User Name: Administrator
```

with:

```text
Event ID: 4732
```

---

# 17. Wazuh Discover Investigation

During my investigations, I worked with Wazuh fields including:

```text
agent.name
agent.id
agent.ip

data.win.eventdata.commandLine
data.win.eventdata.currentDirectory

data.win.eventdata.targetUserName
data.win.eventdata.targetDomainName

data.win.eventdata.subjectUserName
data.win.eventdata.subjectDomainName

data.win.system.eventID
data.win.system.message

rule.id
rule.description
rule.mitre.id
```

I used these fields to filter events and inspect information generated by my monitored systems.

---

# 18. Linux Agent Investigation

I also worked with my Linux Wazuh agent.

The screenshots showed the agent connected to Wazuh and I investigated SSH session activity.

I searched for SSH session events and inspected logs including:

```text
pam_unix(sshd:session)
```

and:

```text
session closed for user ubuntu
```

---

# What I Learned From This Project

Through this project, I learned how to:

* Configure Wazuh agents.
* Re-register a Wazuh Windows agent.
* Modify `ossec.conf`.
* Configure File Integrity Monitoring.
* Monitor files on Windows and Linux.
* Troubleshoot Wazuh agent configuration issues.
* Configure custom Wazuh rules.
* Test a detection by generating Windows activity myself.
* Investigate Windows Security Event IDs in Wazuh.
* Use Wazuh Discover to filter and inspect event fields.
* Investigate account creation.
* Investigate account deletion.
* Investigate successful logons.
* Investigate group membership changes.
* Check Sysmon on Windows.
* Check Sysmon on Linux.
* Configure firewall access for Wazuh agent communication.
* Create a SOC dashboard.

---

# Screenshots

The screenshots in this repository are organized according to the work they demonstrate:

```text
screenshots/
│
├── dashboard/
├── wazuh-agent/
├── file-integrity-monitoring/
├── custom-rules/
├── guest-account-detection/
├── windows-event-investigation/
├── sysm
```

I use screenshots as evidence of the work documented in this repository instead of uploading them randomly without context.

---

# Security and Sensitive Information

Before publishing this project, I removed or redacted:

* Passwords.
* Server IP addresses.
* Public IP addresses.
* Sensitive infrastructure details.
* Any information that could unnecessarily expose my lab environment.

The purpose of this repository is to document what I built, configured, tested, and investigated while keeping sensitive information private.
