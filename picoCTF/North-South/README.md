# picoCTF — North-South Writeup

**Category:** Web Exploitation  
**Difficulty:** Medium  
**Points:** 100  
**Author:** Darkraicg492  

---

## Challenge Description

A web server uses geo-based routing to serve different content based on the
visitor's geographic location. Only requests from Iceland (country code `IS`)
are routed to the server holding the flag. Everyone else gets a dead end.

---

## Nginx Configuration Analysis

```nginx
geoip2 /etc/nginx/GeoLite2-Country.mmdb {
    $geoip2_data_country_code default=ZZ country iso_code;
}

upstream north { server 127.0.0.1:8000; }  # no flag
upstream south { server 127.0.0.1:9000; }  # flag here

location / {
    if ($geoip2_data_country_code = IS) {
        proxy_pass http://south;            # Iceland → flag
    }
    proxy_pass http://north;                # everyone else → nothing
}
```

The server uses the GeoIP2 database to look up the country code of the
connecting IP. If the code is `IS` (Iceland) it proxies to port 9000 where
the flag lives. All other IPs hit port 8000 which returns "No flag in this region".

---

## What Didn't Work

Spoofing headers like `X-Forwarded-For`, `X-Real-IP`, and `CF-Connecting-IP`
with Icelandic IPs had no effect — the nginx config reads the real connecting
IP directly via GeoIP2 and does not trust forwarded headers.

---

## Solution

Used **UrbanVPN** (free browser extension) connected to an Iceland - Reykjavik
server. This routes the actual TCP connection through an Icelandic IP
(`37.235.49.61`), which the GeoIP2 database correctly identifies as `IS`,
triggering the flag route.

1. Install UrbanVPN browser extension
2. Select Iceland as the VPN location
3. Connect and visit the challenge URL
4. Flag is served directly on the homepage

---

## Flag
picoCTF{g30_b453d_r0u71n9_59be682b}

---

## Summary

| Field | Value |
|-------|-------|
| Vulnerability | GeoIP bypass via VPN |
| Method | Free Iceland VPN (UrbanVPN) |
| Restricted region | Iceland (IS) |
| Flag | `picoCTF{g30_b453d_r0u71n9_59be682b}` |
