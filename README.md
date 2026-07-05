# Log-Analysis
# Log Analysis Reading Attack Footprints from Defender's Perspective

## Objective
Analyze system logs on a compromised machine to understand what attacks look like from a Blue Team perspective and identify indicators of compromise (IOCs).

## Lab Setup
- Attacker machine: Kali Linux (VMware)
- Target machine: Metasploitable 2 (VMware)
- Access method: SSH into Metasploitable after exploitation
- Logs analyzed: /var/log/auth.log and /var/log/syslog

---

## Commands Used

```bash
# Connect to target
ssh -oHostKeyAlgorithms=+ssh-rsa -oCiphers=+aes128-cbc,3des-cbc msfadmin@192.168.245.129

# Read last 50 lines of auth log
tail -50 /var/log/auth.log

# Watch logs update in real time during attack
tail -f /var/log/auth.log

# Filter specific events
grep "root" /var/log/auth.log
grep "ftp" /var/log/syslog
grep "error" /var/log/auth.log
```

---

## What I Found in auth.log

**Entry 1 — Critical Finding:**
```
session opened for user root
session closed for user root
```

**Why this is suspicious:**
- Root session appeared with no preceding login event
- No msfadmin or any user authenticated before root appeared
- No legitimate reason for unexpected root session
- No msfadmin or secondary account escalated privileges via su or sudo before root appeared.

---

## What I Found in syslog

**Entry 1 — Normal but relevant:**
```
listening on IPv4 interface
```
Services running and accepting connections — confirms vsftpd was active on port 21.

**Entry 2 — Noteworthy:**
```
Bind to port 22 on 0.0.0.0 failed: Address already in use
```
SSH service startup conflict — harmless in this context but worth investigating in real environments as it could indicate malware attempting to start a backdoor SSH service.

---

## Complete Attack Timeline Reconstructed From Logs

```
/var/log/syslog   → vsftpd running on port 21
/var/log/auth.log → FTP anonymous login allowed
/var/log/auth.log → session opened for user root ← exploit triggered
/var/log/auth.log → session closed for user root ← attacker exited
```

---

## Why No msfadmin in Logs

The vsftpd 2.3.4 backdoor completely bypasses the authentication system:

```
Normal login path:
User enters credentials → auth system validates → session created

Backdoor path:
:) trigger sent → port 6200 opens → root shell → auth system skipped entirely
```

This makes it significantly more dangerous — there is no username to trace, no failed attempts before success, no authentication event whatsoever. Root simply appears.

---

## SOC Analyst Perspective — What Would Trigger Alerts

| Log Entry | Severity | Reason |
|---|---|---|
| Root session with no prior login | Critical | Authentication bypass indicator |
| Anonymous FTP login | High | Unauthorized access vector |
| Port 6200 connection after FTP | Critical | Known vsftpd backdoor indicator |

---

## SIEM Detection Rule This Would Trigger

```
IF session opened for root
AND no preceding authentication event
AND prior FTP activity on port 21
THEN alert: Possible vsftpd backdoor exploitation
Severity: Critical
```

---

## Key Takeaways

1. Auth.log is the first place to check during any incident
2. Root sessions appearing without preceding logins = immediate red flag
3. Backdoor exploits leave minimal but detectable traces
4. Correlating multiple log files tells the complete attack story
5. tail -f is essential for real time monitoring during active incidents
6. syslog and auth.log together give full picture of system and authentication events

---

## Tools & Commands Reference

| Command | Purpose |
|---|---|
| tail -f | Live log monitoring |
| tail -50 | Last 50 entries |
| grep "root" | Filter root events |
| grep "Failed" | Filter failed logins |
| cat /var/log/auth.log | Full log view |

## Additional Log Analysis Findings

### Finding 1 — Application Layer Logs (syslog)

**Command used:**
cat /var/log/syslog | tail -30

**What I found:**
Apache Tomcat (jsvc.exec) running on Metasploitable
with load balancer rules active. This correlates
directly with Nmap scan results showing:
- Port 8009 (ajp13) 
- Port 8180 (Tomcat HTTP)

**Key lesson:**
Syslog and Nmap results tell the same story from
different angles. Cross-referencing both confirms
which services are genuinely running vs just open ports.

---

### Finding 2 — Error Log Analysis (grep "error")

**Command used:**
grep "error" /var/log/syslog

**What I found:**
Three types of errors appearing repeatedly:

1. ACPI DSDT not found (kernel)
   → VM artifact, no real hardware firmware
   → Harmless, appears on every boot

2. Java IOException (Tomcat)
   → Trying to reach dead URL (jakarta.apache.org)
   → Harmless, optional config file missing

3. xinetd loading vsftpd configuration
   → Service manager loading the exact service
      I exploited via CVE-2011-2523
   → Confirms vsftpd was loaded and active

**Key lesson:**
Most log entries are noise. A SOC analyst's core
skill is separating harmless errors from real
indicators of compromise. Out of everything grep
returned, only the xinetd/vsftpd line was
actually relevant to the attack.

---

### Blue Team Takeaway — Signal vs Noise

| Log Entry | Threat? | Why |
|---|---|---|
| ACPI DSDT error | No | VM hardware artifact |
| Java IOException | No | Dead configuration URL |
| xinetd loading vsftpd | Relevant | Confirms attack surface |
| Root session no login | Critical | Authentication bypass |

---
*All analysis performed on owned Metasploitable 2 VM in isolated home lab environment.*
<img width="1676" height="390" alt="error log" src="https://github.com/user-attachments/assets/26525fb7-c46c-449c-8f33-e2d14d4b7539" />
<img width="1337" height="257" alt="ftp log" src="https://github.com/user-attachments/assets/b76ade5a-0aa4-4b08-89d8-4064586fc5b5" />
<img width="1337" height="352" alt="logs" src="https://github.com/user-attachments/assets/31e18e95-afae-4e05-a504-e1d54a4d637e" />
<img width="1290" height="767" alt="msfconsole" src="https://github.com/user-attachments/assets/db71400e-05b5-4859-ab1d-559d82deccfe" />
